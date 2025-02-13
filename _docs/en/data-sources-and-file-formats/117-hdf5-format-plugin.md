---
title: "HDF5 Format Plugin"
slug: "HDF5 Format Plugin"
parent: "Data Sources and File Formats"
---

**Introduced in release:** 1.18.

Hierarchical Data Format (HDF) is a set of file formats designed to store and organize large amounts of data [^1].  Originally developed at the National Center for Supercomputing Applications, it is supported by The HDF Group, a non-profit corporation whose mission is to ensure continued development of HDF5 technologies and the continued accessibility of data stored in HDF [^2].

This plugin enables Apache Drill to query HDF5 files.

## Configuring the HDF5 Format Plugin

There are three configuration variables in this plugin and which are tabled below.

| Option      | Default | Description                                                                                                                                                     |
|-------------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| type        | (none)  | Set to "hdf5" to make use of this plugin                                                                                                                        |
| extensions  | ".h5"   | This is a list of the file extensions used to identify HDF5 files. Typically HDF5 uses `.h5` or `.hdf5` as file extensions.                                     |
| defaultPath | null    | The default path defines which path Drill will query for data. Typically this should be left as `null` in the configuration file. Its usage is explained below. |
|-------------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|

### Example Configuration

For most uses, the configuration below will suffice to enable Drill to query HDF5 files.

    "hdf5": {
          "type": "hdf5",
          "extensions": [
            "h5"
          ],
          "defaultPath": null
        }

## Usage

Since HDF5 can be viewed as a file system within a file, a single file can contain many datasets. For instance, if you have a simple HDF5 file, a star query will produce the following result:

    apache drill> select * from dfs.test.`dset.h5`;
    |-------|-----------|-----------|-----------|---------------|--------------|------------------|-------------------|------------|--------------------------------------------------------------------------|
    | path  | data_type | file_name | data_size | element_count | is_timestamp | is_time_duration | dataset_data_type | dimensions | int_data                                                                 |
    |-------|-----------|-----------|-----------|---------------|--------------|------------------|-------------------|------------|--------------------------------------------------------------------------|
    | /dset | DATASET   | dset.h5   | 96        | 24            | false        | false            | INTEGER           | [4, 6]     | [[1,2,3,4,5,6],[7,8,9,10,11,12],[13,14,15,16,17,18],[19,20,21,22,23,24]] |
    |-------|-----------|-----------|-----------|---------------|--------------|------------------|-------------------|------------|--------------------------------------------------------------------------|

The actual data in this file is mapped to a column called int_data. In order to effectively access the data, you should use Drill's `FLATTEN()` function on the `int_data` column, which produces the following result.


apache drill> select flatten(int_data) as int_data from dfs.test.`dset.h5`;

    |---------------------|
    | int_data            |
    |---------------------|
    | [1,2,3,4,5,6]       |
    | [7,8,9,10,11,12]    |
    | [13,14,15,16,17,18] |
    | [19,20,21,22,23,24] |
    |---------------------|

Once the data is in this form, you can access it similarly to how you might access nested data in JSON or other files. 

    apache drill> SELECT int_data[0] as col_0,
    . .semicolon> int_data[1] as col_1,
    . .semicolon> int_data[2] as col_2
    . .semicolon> FROM ( SELECT flatten(int_data) AS int_data
    . . . . . .)> FROM dfs.test.`dset.h5`
    . . . . . .)> );
    |-------|-------|-------|
    | col_0 | col_1 | col_2 |
    |-------|-------|-------|
    | 1     | 2     | 3     |
    | 7     | 8     | 9     |
    | 13    | 14    | 15    |
    | 19    | 20    | 21    |
    |-------|-------|-------|

However, a better way to query the actual data in an HDF5 file is to use the `defaultPath` field in your query. If the `defaultPath` field is defined in the query, or via the plugin configuration, Drill will only return the data, rather than the file metadata.
 
{% include startnote.html %} Once you have determined which data set you are querying, it is advisable to use this method to query HDF5 data. {% include endnote.html %}
 
{% include startnote.html %} Datasets larger than 16MB will be truncated in the metadata view. {% include endnote.html %}
 
You can set the `defaultPath` variable in either the plugin configuration, or at query time using the `table()` function as shown in the example below:
 

    SELECT * 
    FROM table(dfs.test.`dset.h5` (type => 'hdf5', defaultPath => '/dset'))

 This query will return the result below:
 

    apache drill> SELECT * FROM table(dfs.test.`dset.h5` (type => 'hdf5', defaultPath => '/dset'));
    |-----------|-----------|-----------|-----------|-----------|-----------|
    | int_col_0 | int_col_1 | int_col_2 | int_col_3 | int_col_4 | int_col_5 |
    |-----------|-----------|-----------|-----------|-----------|-----------|
    | 1         | 2         | 3         | 4         | 5         | 6         |
    | 7         | 8         | 9         | 10        | 11        | 12        |
    | 13        | 14        | 15        | 16        | 17        | 18        |
    | 19        | 20        | 21        | 22        | 23        | 24        |
    |-----------|-----------|-----------|-----------|-----------|-----------|
    4 rows selected (0.223 seconds)


If the data in `defaultPath` is a column, the column name will be the last part of the path. If the data is multidimensional, the columns will get a name of `<data_type>_col_n`
. Therefore a column of integers will be called `int_col_1`.

### Attributes

Occasionally, HDF5 paths will contain attributes. Drill will map these to a map data structure called `attributes`, as shown in the query below.

    apache drill> SELECT attributes FROM dfs.test.`browsing.h5`;
    |-------------------------------------------------------------------------------------------------------------------------------|
    | attributes                                                                                                                    |
    |-------------------------------------------------------------------------------------------------------------------------------|
    | {}                                                                                                                            |
    | {"__TYPE_VARIANT__":"TIMESTAMP_MILLISECONDS_SINCE_START_OF_THE_EPOCH"}                                                        |
    | {}                                                                                                                            |
    | {}                                                                                                                            |
    | {"important":false,"__TYPE_VARIANT__timestamp__":"TIMESTAMP_MILLISECONDS_SINCE_START_OF_THE_EPOCH","timestamp":1550033296762} |
    | {}                                                                                                                            |
    | {}                                                                                                                            |
    | {}                                                                                                                            |
    |-------------------------------------------------------------------------------------------------------------------------------|
    8 rows selected (0.292 seconds)

You can access the individual fields within the `attributes` map by using the structure `table.map.key`. Note that you will have to give the table an alias for this to work properly.

    apache drill> SELECT path, data_type, file_name
    FROM dfs.test.`browsing.h5` AS t1 WHERE t1.attributes.important = false;
    |---------|-----------|-------------|
    | path    | data_type | file_name   |
    |---------|-----------|-------------|
    | /groupB | GROUP     | browsing.h5 |
    |---------|-----------|-------------|


### Known Limitations

There are several limitations of the HDF5 format plugin in Drill.

* Drill cannot read unsigned 64 bit integers. When the plugin encounters this data type, it will write an INFO message to the log.
* While Drill can read compressed HDF5 files, Drill cannot read individual compressed fields within an HDF5 file.
* HDF5 files can contain nested data sets of up to `n` dimensions. Since Drill works best with two dimensional data, datasets with more than two dimensions are reduced to 2
 dimensions.
 * HDF5 has a `COMPOUND` data type. At present, Drill supports reading `COMPOUND` data types that contain multiple datasets. At present Drill does not support `COMPOUND` fields
  with multidimesnional columns. Drill will ignore multidimensional columns within `COMPOUND` fields.
 
[^1]: https://en.wikipedia.org/wiki/Hierarchical_Data_Format
[^2]: https://www.hdfgroup.org
