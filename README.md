# Traffic Violence in Chicago: Crashes Involving Pedestrians & Cyclists
### Project Motivation
Traffic violence is a critical public safety issue in the United States. In 2021 alone, approximately [42,915 Americans](https://www.cnbc.com/2022/05/17/us-traffic-deaths-hit-16-year-high-in-2021-dot-says.html) were killed by cars, the most since 2005. However, these numbers should not shock us, as it is well documented that dangerous driving has been on the rise during the pandemic years, even finally catching the attention of national media ([1](https://www.latimes.com/world-nation/story/2021-12-08/traffic-deaths-surged-during-covid-19-pandemic-heres-why), [2](https://www.nytimes.com/2022/02/14/us/pedestrian-deaths-pandemic.html)). When traffic virtually disappeared at the beginning of the pandemic, overbuilt American roads designed to prioritize vehicular throughput above all else suddenly felt more like race tracks than streets, and drivers simply behaved the way environmental cues like wide lanes and few traffic calming features like street trees or tall buildings implicitly suggest, which is to go fast. Dangerous driving behaviors have remained even as the pandemic wanes and streets have filled back up with people. Chicago is no exception to this trend: last year road fatalities spiked to [173 deaths](https://chi.streetsblog.org/2022/07/12/data-road-width-isnt-to-blame-for-chicagos-racial-disparities-in-speed-camera-ticketing/) in the city.

Vulnerable road users like pedestrians and cyclists have been disproportionately affected by dangerous driving, [with pedestrian deaths up 13%](https://www.cnbc.com/2022/05/17/us-traffic-deaths-hit-16-year-high-in-2021-dot-says.html) in 2021 from the previous year, and [cyclist deaths rising 16% in 2020 and another 5% in 2021](https://www.npr.org/2022/05/25/1099566472/more-cyclists-are-being-killed-by-cars-advocates-say-u-s-streets-are-the-problem). In summer 2022 alone, [motorists have struck and killed 6 Chicago children](https://chi.streetsblog.org/2022/08/13/chicagos-car-centric-streets-take-the-life-of-another-child-taha-khan-5-in-sauganash/). In order to better understand this public safety crisis, I mapped out where pedestrian and cyclist crashes occur in Chicago and examined features of the built environment at these locations.

### Research questions
RQ1: Where do auto/pedestrian and auto/cyclist crashes occur in Chicago? Are there any intersections, streets, or wards with a high incidence of crashes?
RQ2: Why do auto/pedestrian and auto/cyclist crashes occur, according to police reports?
RQ3: How do various elements of the built environment affect frequency and severity of crashes? Where were road users at the time of the crash?

### Data sources
All data was sourced from the City of Chicago data portal.
1. [Crashes](https://data.cityofchicago.org/Transportation/Traffic-Crashes-Crashes/85ca-t3if)
2. [People](https://data.cityofchicago.org/Transportation/Traffic-Crashes-People/u6pd-qa9d)
3. [Vehicles](https://data.cityofchicago.org/Transportation/Traffic-Crashes-Vehicles/68nd-jvt3)
4. [Street Center Lines](https://data.cityofchicago.org/Transportation/Street-Center-Lines/6imu-meau)
5. [Bike Routes](https://data.cityofchicago.org/Transportation/Bike-Routes/3w5d-sru8)
6. [Ward Boundaries](https://data.cityofchicago.org/Facilities-Geographic-Boundaries/Boundaries-Wards-2015-2023-/sp34-6z76)

More information about traffic crash reporting in Illinois can be found [here](https://idot.illinois.gov/Assets/uploads/files/Transportation-System/Manuals-Guides-&-Handbooks/Safety/Illinois%20Traffic%20Crash%20Report%20SR%201050%20Instruction%20Manual%202019.pdf).

The most recent pedestrian crash contained in the set is from 8/8/2022 at 5:22 PM. The oldest is from 8/11/15 at 6:30 AM.
The most recent cyclist crash contained in the set is from 8/8/2022 at 4:35 PM. The oldest is from 8/18/15 at 1:30 PM.


### Data cleaning
The 'exploratory analysis' Python script is useful for anyone wanting to understand what is contained in these data sets upon download.
The 'data cleaning' Python script is useful for understanding how I filtered out certain data, cleaned the data, and generated new fields.

#### Crashes
In the crashes data set, each row represents a crash.
First, I retrieved the crashes data by making a request to the API. Then, I converted the geojson to a geoDataFrame.
I filtered out crashes where the road_defect field contained the value 'DEBRIS IN ROADWAY' and eliminated crashes where the traffic control device was not working or functioning improperly in order to be able to rule these out as the cause of the crash.

Because there were 3,877 null geometries, I first tried to generate them from the address using geocoding and iterrows. After running this, there were still 2,097 null geometries remaining. Upon further investigation, these were addresses such as '1 W Parking Lot A' and quite a few addresses on O'Hare International Airport grounds. These nonce addresses and O'Hare addresses were eliminated.

The fields 'injuries reported not evident' and 'injuries no indication' were combined into one category called 'injuries none'. These reflect crashes where a pedestrian or cyclist was hit but not injured.

The geoDataFrame was saved as a geojson called crashes_cleaned.geojson.

#### Pedestrians, Cyclists
Each row in the People data set represents a person: a driver, passenger, pedestrian, cyclist, etc. Crashes, People and Vehicles are linked on crash_record_id.
Like crashes, the people data set was retrieved through an API. Two separate dataframes were created, one for pedestrians and one for cyclists. Only pedestrians and cyclists were retrieved from the API using SoQL clauses to delimit the person_type field.

I filtered out both pedestrians and cyclists whose physical_condition was listed as:
IMPAIRED - ALCOHOL',
'HAD BEEN DRINKING',
'IMPAIRED - DRUGS',
'IMPAIRED - ALCOHOL AND DRUGS',
'FATIGUED/ASLEEP',
'ILLNESS/FAINTED',
'MEDICATED'

I also filtered out both pedestrians and cyclists who had INTOXICATED PED/PEDAL as the listed value under the 'pedpedal_action' field.

As in the crashes set, I combined 'INJURIES REPORTED NOT EVIDENT' and 'INJURIES NO INDICATION' into one value called 'NO INJURY'.

Each of the pedestrians and cyclists dataframes were saved to CSV files as peds_cleaned.csv and cyclists_cleaned.csv.

#### Which crashes were included?
Crashes were filtered down to observations where the crash_record_id was contained in peds_cleaned or cyclists_cleaned. An alternative would have been to use the first_crash_type field in crashes, which has values like 'pedestrian' and 'pedalcyclist'. I easily could have just filtered this field for these values. However, one crash could involve multiple pedestrians. In fact, the data shows that there are about 1,400 more pedestrians in the People data set than there are crashes with a first_crash_type of 'pedestrian'.

#### Spatial join

Because I was interested in how the roadway class affects pedestrian and cyclist crashes, I joined the 'roadway_class' field from the Street Center Lines data set to crashes_cleaned.geojson.
To do this, I spatially left joined crashes_peds_df and street center lines, with crashes_peds_df as the left table using geopandas.sjoin_nearest with the max_distance argument set equal to 0.001. I repeated the same join for crashes_cyclists_df.

After augmenting the crashes with the roadway class, I replaced the class codes with their corresponding categories using iterrows.

I saved crashes_peds_df and crashes_cyclists_df as geojson files.

### Data visualization

After creating connections to all relevant data in Tableau, I created a relationship between crashes_peds_df and peds_cleaned and between crashes_cyclists_df and cyclists_cleaned.
I created the ward and bike route map layers by joining (geometry intersects geometry) them to crashes_peds_df and crashes_cyclists_df.

The field 'Incapacitating injuries' was given the alias 'serious injuries'. This category denotes severe lacerations, broken limbs, skull, chest or abdominal injuries.
The field 'Non Incapacitating injuries' was given the alias 'moderate injuries'. It includes abrasions, bruises, minor lacerations, and lumps on the head.
These explanations can be found on the page for 'Crashes' on the data portal.

### Caveats and clarifications

#### Data cleaning
The data was not normalized for traffic volume or population density. The latest publicly available traffic volume data for the city of Chicago is [here](https://data.cityofchicago.org/Transportation/Average-Daily-Traffic-Counts-Map/pf56-35rv). Not only is the information outdated, but normalizing incidents of traffic violence by traffic volume has an insidious logical underpinning, which is that more people are going to get hurt on busier roads. It doesn't have to be this way, and simply isn't in many other global cities. In places like [Oslo and Helsinki](https://www.theguardian.com/world/2020/mar/16/how-helsinki-and-oslo-cut-pedestrian-deaths-to-zero), they have gone entire calendar years with zero traffic deaths for any type of road user.

#### Tableau

Wrong roadway class?
There are times when the tool tip on the map might say something like “1000 N Cicero Ave” but say “other streets” for roadway class even though Cicero Avenue is an arterial road at that location. The reason for the discrepancy is as follows: The address is a field populated with what the police officer put in the report. The roadway class is a field generated using longitude and latitude, where the longitude and latitude was a point geometry that was spatially joined (gpd.sjoin_nearest) to a multilinestring from the Street Center Lines dataset. The multilinestrings each contained a value for the roadway class, so the join returned a roadway class for each point geometry. The max_distance argument in sjoin_nearest was set equal to 0.001. This max distance was determined through trial and error until all point geometries had non-null values after the spatial join. All that to say, while the roadway class is matched to the long/lat, it may not be an exact match with the address. In the case of "1000 N Cicero Ave" and roadway class, "other streets" refers to Augusta Blvd. A look up of the long/lat of this crash (-87.74604, 41.89873) also suggests it may have happened on August Blvd, not Cicero Ave. Finally, the utility of the roadway class field also becomes less relevant for crashes that occurred at intersections.

Where is x crash I heard about?
If there is a specific crash you’re expecting to see in the data but don’t, there are many reasons it might not be there. For an overview of what data was included, refer to the “data cleaning” section above. It’s also worth mentioning that while large datasets like these are useful for seeing trends, they often aren’t great for looking at individual occurrences of a given phenomenon. For instance, the killing of cyclist Joshua Avina-Luna on 6/24/22 does not appear as a “fatality” in this dataset because he was pronounced dead at a hospital a few days later, not at the time and place of the crash. The same is true for other cases like that of Gerardo Marciales and Paresh Chhatrala. Large analyses like this one are not great at capturing this, nor is it their intended purpose.

Whose fault?
Primary and secondary contributory causes are entered per crash, not per unit (units are motor or non-motor vehicles, including pedestrians and cyclists). Units that are "known or perceived" to be at fault are listed as unit_no '1' in Vehicles. If fault cannot be determined, the "striking unit" is listed as unit_no '1'. In the pedestrian crashes included here, 1,131 of 14,389 have the pedestrian listed as unit_no '1'. In the cyclist crashes included here, 2,626 of 9,292 have the cyclist listed as unit_no '1'. Unfortunately, with current reporting practices, we cannot be sure if these are cases where that unit was "at fault" or simply the "striking" unit. With pedestrians, it seems unlikely that they could be considered the "striking unit", though with cyclists this is more plausible.

### Findings and conclusions
RQ1
Pedestrian and cyclist crashes occur all over the city.

Where do pedestrian fatalities occur?
Since 2015, the majority of pedestrian fatalities have occured in areas where the median household income was less than $51,200 (2018). These correspond to majority Hispanic and Black parts of the city. The reasons behind this fact are manifold and complex, but primary among them are how physical and social factors combine to cause more driving and more speeding in these areas. They include environmental factors like [density, vacancy, congestion, and land use,](https://chicago.suntimes.com/2022/7/19/23270237/debate-over-speeding-tickets-misses-larger-point-about-traffic-safety) and social factors like [jobs  per  household,  children  per  household,  percent  multi-person  households, and household  income,](https://uofi.app.box.com/s/xz94sfxhstivi1r0pu0rzl11mfm3s3oh) among others.

Where do cyclist fatalities occur?
Cyclist fatalities are less tied to economics and race than pedestrian fatalities. Many cyclist crashes occur along official bike routes, which suggests that current infrastructure of painted lines is not adequate in protecting cyclists. We also see certain types of bike routes that may be better than others. For instance, neighborhood greenways have fewer crashes than other types of bike routes.

RQ2
The top category for both pedestrian and cyclist crashes is “unable to determine”, followed by the driver failing to yield the right-of-way. The first takeaway here is that cause documentation is not robust in pedestrian and cyclist crashes. We need better data collection methods if we are going to understand why these crashes occur, for example, officers being required to record something more informative than “unable to determine” except under very special circumstances. A second key takeaway is this: The fact that the top causes for these crashes is that we don’t know, or a cause that has to do with driver behavior, points to another root cause altogether that is not captured in the categories that we have available, which is that of dangerous design. We know that street design is a major factor in how people in cars behave, and right now, the design is causing pedestrians and cyclists who are doing everything right to still get hit.

RQ3
We see that the large majority of the time pedestrians are hit, they are crossing with the signal and using crosswalks. We also see the same data collection problem as before, with categories like “other action” and “unknown” in the top 5. Similarly, cyclists are most often biking with traffic in the roadway, which on most Chicago streets, is the only option available to them. Pedestrians and cyclists are using the infrastructure appropriately, therefore the problem most likely lies with the failed design of the environment.


Thanks to Chris Wright, Joshua Rio Ross, and all the great instructors at NSS. S/o to my DDA7 classmates. We did it!

All mistakes and oversights are my own.
