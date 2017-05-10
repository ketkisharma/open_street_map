# Clustering of Restaurants 

This project is based on data from Open Street Map.
Map Area
[San Francisco, California, United States](https://www.openstreetmap.org/relation/111968)

## Data cleaning
After auditing the sample file for the San Francisco city area, I noticed the following problems in the data related to the features I was interested in exploring:
* Abbreviated street names e.g. St. or St for Street
*	Colon character (:) in tag "k" values
*	The cuisine description for some multi-cuisine restaurants is written as "indian;pakistani" or "chinese;donut"
*	Some of the cuisine names have underscores in the name such as "coffee_shop" or "ice_cream"
*	Five of the rows in the nodes file did not have user information associated with the node ids, which led to an error while uploading the file to the database. 
### Abbreviated Street Names
On auditing the data for street names, various abbreviations for common street descriptions were observed such as - 
'Ave': set(['Esplanade Ave',
             'Lorton Ave',
             'Magnolia Ave',
             'Pennsylvania Ave',
             'Tehama Ave',
             'Telegraph Ave']),
 'Dr': set(['Chateau Dr']),
 'St': set(['Laurel St', 'Leavenworth St', 'Park St']),

The following function was used in the code to update the street names- 
def update_street_name(name): 
    mapping = { "St": "Street", "St.": "Street", "Rd.": "Road", "Rd": "Road", "Ave": "Avenue", "Dr": "Drive"} 
    words = name.split() 
    for i in range(len(words)): 
        if words[i] in mapping: 
            words[i] = mapping[words[i]] 
    return ' '.join(words) 


### Colons in tag values
Some of the tag "k" values which contain address information contain a colon (:) character in between. For example - "addr:street"
If the tag "k" value contained a colon character the characters before the colon were set as the tag type and characters after the colon set as the tag key. If there were additional colon characters in the "k" value, they were ignored and kept as part of the tag key. The following function was used in the code for this purpose - 
values = tag.get('v').split(';') 
        for value in values: 
            parsed_tag = {} 
            parsed_tag['id'] = parent_id 
            k_value = tag.get('k') 
            idx_colon = k_value.find(':') 
            if idx_colon == -1: 
                # Parse for k without ':'  
                parsed_tag['type'] = default_tag_type 
                parsed_tag['key'] = k_value 
            else: 
                parsed_tag['type'] = k_value[0:idx_colon] 
                parsed_tag['key'] = k_value[idx_colon+1:] 
            parsed_tag['value'] = clean_value(value, parsed_tag['key'], k_value) 
            all_rows.append(parsed_tag) 
        return all_rows 

### Cuisine fields for multi-cuisine restaurants
After the data was imported into the database, some basic querying revealed that some of the multi-cuisine or regular restaurants had cuisine values of the form "indian;pakistani". I am interested in exploring the amenities in the area including the number of restaurants of a particular cuisine. Therefore, I separated the cuisine values which have fields separated by semi-colon into different rows in the nodes_tags csv file (code shown above). 
Cuisine fields with underscores 
Some of the cuisine names had underscores in their names such as latin_american or ice_cream. To make sure all the cuisine names showed up in queries, I removed the underscore characters before converting the data to csv format using the function:
def clean_value(value, key_name, full_key_name): 
    if is_street_name(full_key_name): 
        return update_street_name(value) 
    elif 'cuisine' in key_name.lower(): 
        return value.replace('_', ' ').lower() 
    else: 
        return value 

### Empty user information for some nodes
When the data was converted to csv format and uploaded to database, the nodes file had some NaN values in userid and username which resulted in an error while upload. So I went back and removed the 5 rows corresponding to NaN values using the following function (new nodes file named nodes-cleaned). It is likely that this might have been due to some error in system that user information was not recorded since two of the rows had same timestamps. 
def read_csv(filename, outputfile): 
    rows_to_remove = [53800, 53801, 53804, 53835, 53846] 
    with open(filename, 'rb') as f_read: 
        reader = unicodecsv.DictReader(f_read) 
        with open(outputfile, 'wb') as f_write: 
            writer = unicodecsv.DictWriter(f_write, reader.fieldnames) 
            writer.writeheader() 
            for row_idx, row in enumerate(reader): 
                if row_idx not in rows_to_remove: 
                    writer.writerow(row) 
                
## Exploring the Data 

File sizes
san-francisco.osm ......... 1278.5 MB 
data_cleaned.db ........... 883 MB 
nodes-cleaned.csv ......... 503 MB 
nodes_tags.csv ............ 9.2 MB 
ways.csv .................. 44 MB 
ways_tags.csv ............. 57 MB 
ways_nodes.cv ............. 171 MB   

Number of nodes
cur.execute('SELECT COUNT(*) FROM nodes') 
6145349

Number of ways
cur.execute('SELECT COUNT(*) FROM ways') 
751678

Number of unique and top users
Query for number of unique users - 
cur.execute('''
            SELECT COUNT(DISTINCT(x.uid))          
            FROM (SELECT uid FROM nodes UNION ALL SELECT uid FROM ways) x;
            ''')
2662

Query for top 10 users-
cur.execute('''
SELECT uid,  user, count(*) as count_num_nodes
FROM (SELECT nodes.id as id, nodes.uid as uid, nodes.user as user 
    FROM nodes, nodes_tags 
    WHERE nodes.id = nodes_tags.id 
    GROUP BY nodes.id, nodes.uid, nodes.user
   ) as node_user_pair
GROUP BY uid
ORDER BY count_num_nodes DESC
LIMIT 10;
''')

[(933797, u'oba510', 11719),
 (169004, u'oldtopos', 7301),
 (371121, u'AndrewSnow', 6412),
 (1295, u'robert', 5644),
 (153669, u'dchiles', 5482),
 (11154, u'beej71', 4074),
 (481533, u'dbaron', 3720),
 (14293, u'KindredCoda', 3565),
 (28775, u'StellanL', 3467),
 (22925, u'ELadner', 3021)]
  
Number and type of amenities
Total number of restaurants - 

cur.execute('SELECT COUNT(*) FROM nodes_tags WHERE nodes_tags.value= "restaurant" GROUP BY nodes_tags.id')
2889

Total number of restaurants of a particular cuisine (say thai) - 

cur.execute('''
SELECT nodes.lat, nodes.lon
FROM nodes, nodes_tags 
WHERE (nodes.id=nodes_tags.id) AND 
      (nodes_tags.key='cuisine') AND 
      (lower(nodes_tags.value) = "thai")
GROUP BY nodes.id, nodes.lat, nodes.lon
''')
107
   
Total number of coffee shops, cafes or tea places – 

cur.execute('''
SELECT nodes.lat, nodes.lon
FROM nodes, nodes_tags 
WHERE (nodes.id=nodes_tags.id) AND 
      --(nodes_tags.key='cuisine') AND 
      ((lower(nodes_tags.value) like "%coffee%") or (lower(nodes_tags.value) like "%cafe%") or (lower(nodes_tags.value) like "%tea%"))
      GROUP BY nodes.id, nodes.lat, nodes.lon
''')
1381
    
