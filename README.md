# Cluster analysis based on OpenStreetMap data

This project is based on data from Open Street Map for [San Francisco](https://www.openstreetmap.org/relation/111968), California, United States.

## Data cleaning

After auditing the sample file for the San Francisco city area, I noticed the following problems in the data related to the features I was interested in exploring:

* Abbreviated street names e.g. St. or St for Street
* Colon character (:) in tag "k" values
* The cuisine description for some multi-cuisine restaurants is written as "indian;pakistani" or "chinese;donut"
* Some of the cuisine names have underscores in the name such as "coffee_shop" or "ice_cream"
* Five of the rows in the nodes file did not have user information associated with the node ids, which led to an error while uploading the file to the database.

__Abbreviated Street Names__

On auditing the data for street names, various abbreviations for common street descriptions were observed such as - 'Ave': set(['Esplanade Ave', 'Lorton Ave', 'Magnolia Ave', 'Pennsylvania Ave', 'Tehama Ave', 'Telegraph Ave']), 'Dr': set(['Chateau Dr']), 'St': set(['Laurel St', 'Leavenworth St', 'Park St']),

The update_street_name function was used in the code to update the street names.

__Colons in tag values__

Some of the tag "k" values which contain address information contain a colon (:) character in between. 
For example - "addr:street" If the tag "k" value contained a colon character the characters before the colon were set as the tag type and characters after the colon set as the
tag key. If there were additional colon characters in the "k" value, they were ignored and kept as part of the tag key. 

__Cuisine fields for multi-cuisine restaurants__

After the data was imported into the database, some basic querying revealed that some of the multi-cuisine or regular restaurants had cuisine values of the form "indian;pakistani". I am interested in exploring the amenities in the area including the number of restaurants of a particular cuisine. Therefore, I separated the cuisine values which have fields separated by semi-colon into different rows in the nodes_tags csv file.

__Cuisine fields with underscores__

Some of the cuisine names had underscores in their names such as latin_american or ice_cream. To make sure all the cuisine names showed up in queries, I removed the underscore characters before converting the data to csv format using the function clean_value in the code. 

__Empty user information for some nodes__

When the data was converted to csv format and uploaded to database, the nodes file had some NaN values in userid and username which resulted in an error while upload. So I went back and removed the 5 rows corresponding to NaN values using the following function (new nodes file named nodes-cleaned). It is likely that this might have been due to some error in system that user information was not recorded since two of the rows had same timestamps. def read_csv(filename, outputfile): rows_to_remove = [53800, 53801, 53804, 53835, 53846] with open(filename, 'rb') as f_read: reader = unicodecsv.DictReader(f_read) with open(outputfile, 'wb') as f_write: writer = unicodecsv.DictWriter(f_write, reader.fieldnames) writer.writeheader() for row_idx, row in enumerate(reader): if row_idx not in rows_to_remove: writer.writerow(row)

## Exploring the Data

__File sizes__

 san-francisco.osm ......... 1278.5 MB
 
 data_cleaned.db ........... 883 MB
 
 nodes-cleaned.csv ......... 503 MB
 
 nodes_tags.csv ............ 9.2 MB
 
 ways.csv .................. 44 MB
 
 ways_tags.csv ............. 57 MB
 
 ways_nodes.cv ............. 171 MB
 

__Number of nodes__

SELECT COUNT(*) FROM nodes;
6145349

__Number of ways__

SELECT COUNT(*) FROM ways;
751678

__Number of unique and top users__

Query for number of unique users 
SELECT COUNT(DISTINCT(x.uid))
FROM (SELECT uid FROM nodes UNION ALL SELECT uid FROM ways) x; 
2662

__Number and type of amenities Total number of restaurants__

SELECT COUNT(*) FROM nodes_tags WHERE nodes_tags.value= "restaurant" GROUP BY nodes_tags.id;
2889

__Total number of restaurants of a particular cuisine (say thai)__

SELECT nodes.lat, nodes.lon FROM nodes, nodes_tags WHERE (nodes.id=nodes_tags.id) AND (nodes_tags.key='cuisine') 
AND (lower(nodes_tags.value) = "thai") GROUP BY nodes.id, nodes.lat, nodes.lon;
107

## Cluster Analysis

Finally, I have explored the extent to which different types of amenities (restaurants of a particular cuisine, gas stations) cluster together geographically. Based on the theory of [agglomeration of businesses](http://www.economist.com/node/14292202), I have explored the level of clustering of different types of restaurants. 
I implemented the DBSCAN algorithm which can handle arbitrary distance function and clusters a spatial data set based on two parameters: a physical distance from each point, and a minimum cluster size. Based on cluster analysis, the more "exclusive" cuisine restaurants (French, Indian, Italian) show a higher level of clustering than generic ones such as fast food joints and cafes. Amenities such as schools, libraries, places of worship showed much lower level of clustering.

