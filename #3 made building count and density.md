```bash
cd C:\Program Files\osm
-c -d lesotho -K -H localhost --slim -P 5432 -S default.style 2910-lesotho-latest.osm.pbf

The download link I used is: download.geofabrik.de
The date of the download is : 29th October 2016

There is a total of **[1623] mapped buildings for Semonkong**


#A QUERY FOR BUILDINGS

```bash
SELECT building, Count(*)
FROM Semonkong_Inner_Boundary
WHERE building is not null
GROUP BY building
```
#RESULTS
|**building**   |**count**|
|---------------|---------|
|1.	Shed        |	11      |
|2.	industrial 	|1        |
|3.	construction|78       |
|4.	yes	        |1394     | 
|5.	hut        	|137      |
|6.	residential |2        |
|**total**      |**1623** |
 
 To calculate building density, I first had to calculate the area of the polygon: there are two ways in which I can find the area. I did both:

```bash
1.	Simple just click on your polygon or the edge of your polygon (for some PCs) in geojson.io (note that this one does not have the hectares so you might want to convert manually)
After using this method the area of my polygon is:
```
|**unit**       |**Area**    |
|---------------|------------|
|Sq. Meters	    |5714331.44  |
|Sq. Kilometers	|5.71        |
|Sq. Feet     	|61508578.25 |
|Acres	        |1412.04     |
|Sq. Miles	    |2.21        |

            #OR#
```bash
2.	To get the hectares you can install and use npm 
```

After using this method the area of my polygon is:
**571.4331444965953 hectares**
So, in order for me to get the building density, **(total number of buildings per area of my polygon):
1623 / 571.4331444965953 = 2.84022727 buildings per hectares**
