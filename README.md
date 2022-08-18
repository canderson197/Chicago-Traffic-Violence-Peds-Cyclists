# capstone_CA
### Project Motivation
Traffic violence is a critical public safety issue in the United States. In 2020 alone, more than [40,000 Americans](https://www.vox.com/22675358/us-car-deaths-year-traffic-covid-pandemic) were killed by cars. Furthermore, dangerous driving has been on the rise during the pandemic years, even finally catching the attention of mainstream media ([1](https://www.latimes.com/world-nation/story/2021-12-08/traffic-deaths-surged-during-covid-19-pandemic-heres-why), [2](https://www.nytimes.com/2022/02/14/us/pedestrian-deaths-pandemic.html)). The obvious effect of this is that traffic violence has risen as well: last year in Chicago, road fatalities spiked to [173 deaths](https://chi.streetsblog.org/2022/07/12/data-road-width-isnt-to-blame-for-chicagos-racial-disparities-in-speed-camera-ticketing/). This summer, in 2022, [motorists have struck and killed 6 Chicago children](https://chi.streetsblog.org/2022/08/13/chicagos-car-centric-streets-take-the-life-of-another-child-taha-khan-5-in-sauganash/). In order to better understand this public safety crisis, I mapped out where exactly these crashes occur and examined many dimensions of the crashes to determine how road user behavior and the design of the built environment may interact to cause so much traffic violence.


### Data sources
All data was sourced from the City of Chicago data portal.
1. [Crashes](https://data.cityofchicago.org/Transportation/Traffic-Crashes-Crashes/85ca-t3if)
2. [People](https://data.cityofchicago.org/Transportation/Traffic-Crashes-People/u6pd-qa9d)
3. [Street Center Lines](https://data.cityofchicago.org/Transportation/Street-Center-Lines/6imu-meau)
4. [Bike Routes](https://data.cityofchicago.org/Transportation/Bike-Routes/3w5d-sru8)
5. [Ward Boundaries](https://data.cityofchicago.org/Facilities-Geographic-Boundaries/Boundaries-Wards-2015-2023-/sp34-6z76)

More information about traffic crash reporting in Illinois can be found [here](https://idot.illinois.gov/Assets/uploads/files/Transportation-System/Manuals-Guides-&-Handbooks/Safety/Illinois%20Traffic%20Crash%20Report%20SR%201050%20Instruction%20Manual%202019.pdf).

The most recent pedestrian crash contained in the set is from 8/8/2022 at 5:22 PM. The oldest is from 8/11/15 at 6:30 AM. The most recent cyclist crash contained in the set is from 8/8/2022 at 4:35 PM. The oldest is from 8/18/15 at 1:30 PM.


### Data cleaning
The 'exploratory analysis' Python script is useful for anyone wanting to understand what is contained in these data sets upon download.
The 'data cleaning' Python script is useful for seeing how I filtered out certain data, cleaned the data, and generated new fields.

#### Data cleaning - Python

##### Crashes
First, I retrieved the crashes data by making a request to the API. Then, I converted the geojson to a geoDataFrame.
I filtered out crashes where the road_defect field contained the value 'DEBRIS IN ROADWAY' to eliminate these rare cases from the data, and thus possible noise. I eliminated crashes where the traffic control device was not working or functioning improperly in order to be able to rule this out as the cause of the crash.

Because there were 3,877 null geometries, I tried to generate them from the address using geocoding and iterrows():
  <code>for index, row in crashes_df.iterrows():
      if row.geometry is None:
          try:
              geolocator = Nominatim(user_agent="colin")
              location = geolocator.geocode(row['full_address'])
              crashes_df.at[index, 'geometry'] = Point((location.longitude, location.latitude))
          except:
              print(index)<code>

There were still 2,097 null geometries remaining. Upon further investigation, these were addresses such as '1 W Parking Lot A' and quite a few addresses on O'Hare International Airport grounds. These nonce addresses and O'Hare addresses were eliminated.

The fields 'injuries reported not evident' and 'injuries no indication' were combined into one category called 'injuries none'. These reflect crashes where a pedestrian or cyclist was hit but not injured.

The geoDataFrame was saved as a geojson called crashes_cleaned.geojson.

##### Pedestrians, Cyclists

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

Each of the pedestrians and cyclists dataframes were saved to CSV files as peds_cleaned and cyclists_cleaned.

#### Spatial join

Because I was interested in how the roadway class affects pedestrian and cyclist crashes, I joined the 'roadway_class' field from the Street Center Lines data set to crashes_cleaned.geojson.
To do this, I first got the crash_record_id from peds_cleaned and cyclists_cleaned. I filtered crashes_cleaned on these crash_record_ids to create crashes_peds_df and crashes_cyclists_df. I then spatially left joined crashes_peds_df and street center lines with crashes_peds_df as the left table using geopandas.sjoin_nearest with the max_distance argument set equal to 0.001. I repeated the same join for crashes_cyclists_df.

After augmenting the crashes with the roadway class, I replaced the class codes with words using iterrows.

I saved crashes_peds_df and crashes_cyclists_df as geojson files.

#### Data cleaning - Tableau

The field 'Incapacitating injuries' was given the alias 'serious injuries'. This category denotes severe lacerations, broken limbs, skull, chest or abdominal injuries
The field 'Non Incapacitating injuries' was given the alias 'moderate injuries'. It includes abrasions, bruises, minor lacerations, and lumps on the head.
These explanations can be found on the page for 'crashes' on the data portal.

### Data visualization

After creating connections to all relevant data in Tableau, I created a relationship between crashes_peds_df and peds_cleaned and between crashes_cyclists_df and cyclists_cleaned.
I created the ward and bike route map layers by joining (geometry intersects geometry) them to crashes_peds_df and crashes_cyclists_df.

### Caveats and clarifications

#### Dashboard


There are times when the tool tip on the map might say something like “1000 N Cicero Ave” but say “other streets” for roadway class even though Cicero Avenue is an arterial road at that location. The reason for the discrepancy is as follows: The address is a field populated with what the police officer put in the report. The roadway class is a field generated using longitude and latitude, where the longitude and latitude was a point geometry that was spatially joined (gpd.sjoin_nearest) to a multilinestring from the Street Center Lines dataset, where the max distance argument was set equal to 0.001. This max distance was determined through trial and error until all point geometries had non-null values after the spatial join. Therefore, while the roadway class is matched to the long/lat, it may not be an exact match with the address. In the case of "1000 N Cicero Ave" and roadway class, "other streets" refers to Augusta Blvd. A look up of the long/lat of this crash (-87.74604 41.89873) also suggests it may have happened on August Blvd, not Cicero Ave. Finally, the utility of the roadway class field also becomes less relevant for crashes that occurred at intersections.

If there is a specific crash you’re expecting to see in the data but don’t, there are many reasons it might not be there. For an overview of what data was included, refer to the “data cleaning” section of this README. It’s also worth mentioning that while large datasets like these are useful for seeing trends, they often aren’t great for looking at individual occurrences of a given phenomenon. For instance, the killing of cyclist Joshua Avina-Luna on 6/24/22 does not appear as a “fatality” in this dataset because he was pronounced dead at a hospital a few days later, not at the time and place of the crash. The same is true for other cases like that of Gerardo Marciales and Paresh Chhatrala. Large analyses like this one are not great at capturing this, nor is it their intended purpose.

#### Data cleaning
The data was not normalized for traffic volume or population density. The latest publicly available traffic volume data for the city of Chicago is from 2011. Not only is the information outdated, but normalizing (statistically) incidents of traffic violence by traffic volume has an insidious logical underpinning, which is that more people are going to get hurt on busier roads. It doesn't have to be this way, and simply isn't in many other global cities. In places like [Oslo and Helsinki](https://www.theguardian.com/world/2020/mar/16/how-helsinki-and-oslo-cut-pedestrian-deaths-to-zero), they have gone entire calendar years with zero traffic deaths for any type of road user.
