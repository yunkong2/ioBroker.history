![Logo](admin/history.png)
# ioBroker.history
===============================
[![NPM version](http://img.shields.io/npm/v/iobroker.history.svg)](https://www.npmjs.com/package/iobroker.history)
[![Downloads](https://img.shields.io/npm/dm/iobroker.history.svg)](https://www.npmjs.com/package/iobroker.history)

[![NPM](https://nodei.co/npm/iobroker.history.png?downloads=true)](https://nodei.co/npm/iobroker.history/)

This adapter saves state history in a two-staged process.
At first data points are stored in RAM, as soon as they reach maxLength they will be stored on disk.

To set up some data points to be stored they must be configured in admin "Objects" Tab (last button).

## Settings

- **Storage directory** - Path to the directory, where the files will be stored. It can be done relative to "iobroker-data" or absolute, like "/mnt/history" or "D:/History"
- **Maximal number of stored in RAM values** - After this number of values reached in RAM they will be saved on disk.
- **Store origin of value** - If "from" field will be stored too. Can save place on disk.
- **De-bounce interval** - Protection against too often changes of some value.
- **Storage retention** - How many values in the past will be stored on disk.
- **Log unchanged values any(s)** - When using "log changes only" you can set a time interval in seconds here after which also unchanged values will be re-logged into the DB

## Access values from Javascript adapter
The sotred values can be accessed from Javascript adapter. E.g. with following code you can read the list of events for last hour:

```
// Get 50 last stored events for all IDs
sendTo('history.0', 'getHistory', {
    id: '*',
    options: {
        end:       new Date().getTime(),
        count:     50,
        aggregate: 'onchange'
    }
}, function (result) {
    for (var i = 0; i < result.result.length; i++) {
        console.log(result.result[i].id + ' ' + new Date(result.result[i].ts).toISOString());
    }
});

// Get stored values for "system.adapter.admin.0.memRss" in last hour
var end = new Date().getTime();
sendTo('history.0', 'getHistory', {
    id: 'system.adapter.admin.0.memRss',
    options: {
        start:      end - 3600000,
        end:        end,
        aggregate: 'onchange'
    }
}, function (result) {
    for (var i = 0; i < result.result.length; i++) {
        console.log(result.result[i].id + ' ' + new Date(result.result[i].ts).toISOString());
    }
});
```

Possible options:
- **start** - (optional) time in ms - *new Date().getTime()*'
- **end** - (optional) time in ms - *new Date().getTime()*', by default is (now + 5000 seconds)
- **step** - (optional) used in aggregate (m4, max, min, average, total) step in ms of intervals
- **count** - number of values if aggregate is 'onchange' or number of intervals if other aggregate method. Count will be ignored if step is set.
- **from** - if *from* field should be included in answer
- **ack** - if *ack* field should be included in answer
- **q** - if *q* field should be included in answer
- **addId** - if *id* field should be included in answer
- **limit** - do not return more entries than limit
- **ignoreNull** - if null values should be include (false), replaced by last not null value (true) or replaced with 0 (0)
- **aggregate** - aggregate method:
    - *minmax* - used special algorithm. Splice the whole time range in small intervals and find for every interval max, min, start and end values.
    - *max* - Splice the whole time range in small intervals and find for every interval max value and use it for this interval (nulls will be ignored).
    - *min* - Same as max, but take minimal value.
    - *average* - Same as max, but take average value.
    - *total* - Same as max, but calculate total value.
    - *count* - Same as max, but calculate number of values (nulls will be calculated).
    - *none* - No aggregation at all. Only raw values in given period.

The first and last points will be calculated for aggregations, except aggregation "none".
If you manually request some aggregation you should ignore first and last values, because they are calculated from values outside of period.

## Data converter
### Convert History-Data to InfluxDB
The history2influx.js can be found in the directory "converter".

The script will parse directly the generated history JSON files on disk to transfer them into the Database.
Additionally it queries all available Measurements(Datapoints) from the InfluxDB and only converts those, so make sure to enable all logging before and make sure there are data in.

The script will query the earliest timestamp per measurement and will only insert data from History before that date. These earliest-Value information is also cached when the converter is aborted, so that this is not needed to be queried again. With new data migrated from history these data will also be updated.
To reset the data delete the File "earliestDBValues.json".

The Converter then goes backward in time through all the days available as data and will determine which data to transfer to InfluxDB.

If you want to abort the process you can press "x" or "<CTRL-C>" and the converter aborts after the current datafile.

Note: Migrating many data will produce a certain load on the system, especially when converter and target-InfluxDB instance are running on the same machine. Monitor your systems load and performance during the action and maybe use the "delayMultiplicator" to increase delays in the converter.

**Usage:** nodejs history2influx.js [<InfluxDB-Instance>] [<Loglevel>] [<Date-to-start>|0] [<path-to-Data>] [<delayMultiplicator>] [--logChangesOnly [<relog-Interval(m)>]] [--ignoreExistingDBValues]
**Example**: nodejs history2influx.js influxdb.0 info 20161001 /path/to/data 2 --logChangesOnly 30

Possible options and Parameter:
- **<InfluxDB-Instance>**: InfluxDB-Instance to send the data to (Default: influxdb.0). If set needs to be first parameter after scriptname.
- **<Loglevel>**: Loglevel for output (Default: info). If set needs to be second parameter after scriptname.
- **<Date-to-start>**: Day to start in format yyyymmdd (e.g. 20161028). Use "0" to use detected earliest values. If set needs to be third parameter after scriptname.
- **<path-to-Data>**: Path to the datafiles. Defauls to <iobroker-install-directory>/iobroker-data/history-data . If set needs to be fourth parameter after scriptname.
- **<delayMultiplicator>**: Modify the delays between several actions in the script by a multiplicator. "2" would mean that the delays the converted had calculated by itself are doubled. If set needs to be fifth parameter after scriptname.
- **--logChangesOnly [<relog-Interval(m)>]**: when --logChangesOnly is set the data are parsed and reduced, so that only changed values are stored in InfluxDB. Additionally a <relog-Interval(s)> can be set in minutes to re-log unchanged values after this interval.
- **--ignoreExistingDBValues**: With this parameter all existing data are ignored and all data are inserted into DB. Please make sure that no duplicated are generated. This option is usefull to fix "holes" in the data where some data are missing. It still only fills all datapoints that exist in InfluxDB already!

## Changelog
### 1.4.0 (2016-10-29)
* (Apollon77) add option to re-log unchanged values to make it easier for visualization
* (Apollon77) added converter script to move history data to influxdb

### 1.3.1 (2016-09-25)
* (Apollon77) Fixed: ts is assigned as val
* (bluefox) Fix selector for history objects

### 1.3.0 (2016-08-30)
* (bluefox) сompatible only with new admin

### 1.2.0 (2016-08-27)
* (bluefox) change name of object from history to custom

### 1.1.0 (2016-08-27)
* (bluefox) fix aggregation of last point
* (bluefox) aggregation none just deliver the raw data without any aggregation

### 1.0.5 (2016-07-24)
* (bluefox) fix aggregation on large intervals

### 1.0.4 (2016-07-05)
* (bluefox) fix aggregation on seconds

### 1.0.3 (2016-05-31)
* (bluefox) draw line to the end if ignore null

### 1.0.2 (2016-05-29)
* (bluefox) switch max and min with each other

### 1.0.1 (2016-05-28)
* (bluefox) calculate end/start values for "on change" too

### 1.0.0 (2016-05-20)
* (bluefox) change default aggregation name

### 0.4.1 (2016-05-14)
* (bluefox) support sessionId

### 0.4.0 (2016-05-05)
* (bluefox) use aggregation file from sql adapter
* (bluefox) fix the values storage on exit
* (bluefox) store all cached data every 5 minutes
* (bluefox) support of ms

### 0.2.1 (2015-12-14)
* (bluefox) add description of settings
* (bluefox) place aggregate function into separate file to enable sharing with other adapters
* (smiling-Jack) Add generate Demo data
* (smiling-Jack) get history in own fork
* (bluefox) add storeAck flag
* (bluefox) mockup for onchange

### 0.2.0 (2015-11-15)
* (Smiling_Jack) save and load in adapter and not in js-controller
* (Smiling_Jack) aggregation of data points
* (Smiling_Jack) support of storage path

### 0.1.3 (2015-02-19)
* (bluefox) fix small error in history (Thanks on Dschaedl)
* (bluefox) update admin page

### 0.1.2 (2015-01-20)
* (bluefox) enable save&close button by config

### 0.1.1 (2015-01-10)
* (bluefox) check if state was not deleted

### 0.1.0 (2015-01-02)
* (bluefox) enable npm install

### 0.0.8 (2014-12-25)
* (bluefox) support of de-bounce interval

### 0.0.7 (2014-11-01)
* (bluefox) store every change and not only lc != ts

### 0.0.6 (2014-10-19)
* (bluefox) add configuration page

## License

The MIT License (MIT)

Copyright (c) 2014-2015 Bluefox, Smiling_Jack

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
