# Quantum of landuses 

A query that shows both the area of each type of landuse and the types of landuse:
```bash
Select sum (ST_Area (way)) / 1000 area_ha, landuse
From Urban_Council_Polygon
Where landuse is not null
Group By landuse
```
#THE RESULTS
|**Area**            |**landuse**|
|--------------------|-----------|
|0.188017400001474   |Cemetery   |
|51.3395037999519  	 |Commercial |
|10351.5220627495  	 |Farmland   |
|7.38443475000811  	 |Landfill   |
|352.585857249941  	 |Military   |
|17.0787677999725  	 |Retail     |
|5814.04504980008  	 |Residential|
|34.6249139499669  	 |Quarry     |
|6.68926289997522  	 |Industrial |

 All the results can be viewed in Quantum GIS and this is where I will be able to classify and assign RGB values to get the Landuse 
 Classification of Lesotho.
