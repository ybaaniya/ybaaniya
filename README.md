# Earth Observer Tool
This is a Tethys 2/3 compatible app that visualizes NASA GLDAS and NOAA GFS data, and is extendable to other earth observation datasets. This app may be used in conjunction with the GLDAS and GFS Visualizer apps. 

© Riley Hales, 2019. Developed at the BYU Hydroinformatics Lab.

## App Features
1. View time-animated maps of Earth Observation Data. Customize color scheme, opacity, animation/playback speed, time period.
2. View world region boundaries. Customize color scheme and opacity. Choose to get layers from Geoserver or local GeoJSON
3. View the Latitude/Longitude Coordinates of anywhere on the planet by hovering your mouse over the map.
4. Toggle visibility of controls and the controls menu.
5. Use drawing controls to place a point or draw a rectangle on the globe and generate a chart with timeseries values at that point or of the spatial average within the box.
6. Click on a landmass and a pop-up bubble gives you the name of the region and the option to generate a timeseries of average values in that region.
7. Choose between 4 additional chart options using box plots and multi-line plots to view statistical analysis of historical data.
8. Export timeseries charts as graphics.
9. Export timeseries data to csv.
10. App features available through a documented API.

## Installation Instructions
### 1 Install the Tethys App
This application is compatible with Tethys 2.X and Tethys 3 Distributions and is compatible with both Python 2 and 3 and Django 1 and 2. Install the latest version of Tethys before installing this app. This app requires the python packages: numpy, netcdf4, ogr, osr. Both should be installed automatically as part of this installation process.

On the terminal of the server enter the tethys environment with the ```t``` command. ```cd``` to the directory where you install apps then run the following commands:  
~~~~
git clone https://github.com/rileyhales/earthobserver.git  
cd gldas
python setup.py develop
~~~~  
If you are on a production server, run:
~~~~
tethys manage collectstatic
~~~~
Reset the server, then attempt to log in through the web interface as an administrator. The app should appear in the Apps Library page in grey indicating you need to configure the custom settings.

### 2 Set up a Thredds Server (EO Layer Rasters)
Refer to the documentation for Thredds to set up an instance of Thredds on your tethys server.

In the public folder where your datasets are stored, create a new folder called ```earthobserver```. use the ```pwd``` command to get the file path to these datasets and save for future reference.

You will also need to modify Thredds' settings files to enable WMS services and support for netCDF files on your server. In the folder where you installed Thredds, there should be a file called ```catalog.xml```. 
~~~~
vim catalog.xml
~~~~
Type ```a``` to begin editing the document.

At the top of the document is a list of supported services. Make sure the line for wms is not commented out.
~~~~
<service name="wms" serviceType="WMS" base="/thredds/wms/" />
~~~~
Scroll down toward the end of the section that says ```filter```. This is the section that limits which kinds of datasets Thredds will process. We need it to accept .nc, .nc4, and .ncml file types. Make sure your ```filter``` tag includes the following lines.
~~~~
<filter>
    <include wildcard="*.nc"/>
    <include wildcard="*.nc4"/>
    <include wildcard="*.ncml"/>
</filter>
~~~~
Press ```esc``` then type ```:x!```  and press the ```return``` key to save and quit.
~~~~
vim threddsConfig.xml
~~~~
Find the section near the top about CORS (Cross-Origin Resource Sharing). CORS allows Thredds to serve data to servers besides the host where it is located. Depending on your exact setup, you need to enable CORS by uncommenting these tags.
~~~~
<CORS>
    <enabled>true</enabled>
    <maxAge>1728000</maxAge>
    <allowedMethods>GET</allowedMethods>
    <allowedHeaders>Authorization</allowedHeaders>
    <allowedOrigin>*</allowedOrigin>
</CORS>
~~~~
Press ```esc``` then type ```:x!```  and press the ```return``` key to save and quit.

Reset the Thredds server so the catalog is regenerated with the edits that you've made. The command to reset your server will vary based on your installation method, such as ```docker reset thredds``` or ```sudo systemctl reset tomcat```.

### 3 Set up a GeoServer (World Region Boundaries) (Optional, Recommended)
Refer to the documentation for GeoServer to set up an instance of GeoServer on your tethys server. If you choose not to use geoserver, skip this step and follow instructions in step 5 for the custom settings.

This app can display and perform spatial averaging for 8 world regions. The app will perform the raster operations and averaging using shapefiles that cover general regions of the globe in as few points as possible to increase computation speed, reduce file sizes, and prevent computation errors related to large and complex polygon shapefiles. More accurate boundaries for the regions are available for visualization as a Web Feature Service (WFS) through GeoServer or as local geojson files. A copy of the shapefiles you need, in the properly formatted zip archives, is found in the ```workspaces/app_workspace``` directory of the app.   

Use a web browser to log in to your GeoServer. Use the web interface to create a new workspace named ```gldas```. Use the command line to navigate to the directory containing the GeoServerFiles zip archive you got from the app. Extract the contents of that zip archive, but do not unzip the 8 zip archives that it contains. Upload each of those 8 zip archives to the new GeoServer workspace using cURL commands (e.g. run this command 8 times). The general format of the command is:
~~~~
curl -v -u [user]:[password] -XPUT -H "Content-type: application/zip" --data-binary @[name_of_zip].zip https://[hostname]/geoserver/rest/workspaces/[workspaceURI]/datastores/[name_of_zip]/file.shp
~~~~
This command asks you to specify:
* Geoserver Username and Password. If you have not changed it, the default is admin and geoserver.
* Name of the Zip Archive you're uploading. Be sure you spell it correctly and that you put it in each of the 2 places it is asked for.
* Hostname. The host website, e.g. ```tethys.byu.edu```.
* The Workspace URI. The URI that you specified when you created the new workspace through the web interface. If you followed these instructions it should be ```gldas```.

### 4 Set The Custom Settings
Log in to your Tethys portal as an admin. Click on the grey GLDAS box and specify these settings:

**Local File Path:** This is the full path to the directory named gldas that you should have created within the thredds data directory during step 2. You can get this by navigating to that folder in the terminal and then using the ```pwd``` command. (example: ```/tomcat/content/thredds/earthobserver/```)  

**Thredds Base Address:** This is the base URL to Thredds WMS services that the app uses to build urls for each of the WMS layers generated for the netcdf datasets. If you followed the typical configuration of thredds (these instructions) then your base url will look something like ```yourserver.com/thredds/wms/testAll/earthobserver/```. You can verify this by opening the thredds catalog in a web browser (typically at ```yourserver.com/thredds/catalog.html```). Navigate to one of the GLDAS netcdf files and click the WMS link. A page showing an xml document should load. Copy the url in the address bar until you get to the ```/earthobserver/``` folder in that url. Do not include ```/raw/name_of_dataset.nc``` or the request that comes after. (example: ```https://tethys.byu.edu/thredds/wms/testAll/earthobserver/```)

**Geoserver Workspace Address:** This is the WFS (ows) url to the workspace on geoserver where the shapefiles for the world region boundaries are served. This geoserver workspace needs to have at minimum WFS services enabled. You can find it by using the layer preview interface of GeoServer and choosing GeoJSON as the format. If you chose not to use geoserver, enter ```geojson``` as your url. (example: ```https://tethys.byu.edu/geoserver/earthobserver/ows```)

### 5 Get the EO Data
Log in to the app as an administrator. In the menu on the left, there is a user interface for controlling the datasets used by the app. Verify the data shown and use the controls to initiate downloads and processing. Recommended one workflow at a time.

Verify that you have completed these steps 2 and 3 correctly by viewing the Thredds catalog through a web browser. The default address will be something like ```yourserver.com/thredds/catalog.html```. Navigate to the ```Test all files...``` folder. Your ```earthobserver``` folder should be visible. Open it and check that all the additional folders and datasets are visible. If they are not, you either haven't waited for the processes to finish or may need to restart the Thredds Server.

## How To Use The API
  * All api calls are available at a url following the pattern:  
    (tethys_portal_url)/apps/earthobserver/api/(method_name)/(parameter_string)
  * Example parameters (as a python dictionary):  
    ```{'model': 'gfs', 'variable': 'al', 'coords': [-90, 45], 'loc_type': 'Point', 'time': 'alltimes', 'level': 'surface'}```
  * Example API authorization token:  
  ```{'Authorization': 'Token 82ef74ef7a365350996268910315b29c8b1f2272'}```


  ### getcapabilities
  * Parameters: none
  * Returns: JSON response of the available api methods. Specifically,
  ~~~
    {'api_calls': [
        'getcapabilities', 'eodatamodels', 'gldasvariables', 'gldasdates', 'gfsdates',
        'gfslevels','timeseries'
        ]}
  ~~~

  ### eodatamodels
  * Parameters: none
  * Returns: JSON response of a list of lists. Each list contains the full name and description of the EO data and the
    short name which you use in api calls. For examples,
  ```{"models": [["(Historical) NASA GLDAS (Global Land Data Assimilation System)", "gldas"], [etc...] ]```

  ### gldasvariables
  * Parameters: none
  * Returns: JSON response where the keys are the full name of the variable and the values are the shorted name of the variables you will use in api calls
  ```{"Air Temperature": "Tair_f_inst", "Surface Albedo": "Albedo_inst", etc... }```

  ### gldasdates
  * Parameters: none
  * Returns: JSON response describing the earliest and latest dates of your downloaded gldas data as well as instructions for api_calls
  ```{"start": "January 2000", "end": "March 2019", "api_calls": "Provide a string type 4 digit year or "alltimes""}```

  ### gfsdates
  * Parameters: none
  * Returns: JSON response describing the dates and files of available gfs data
  ```{"gfs_time": "Jul 02, 12PM UTC", "gfs_files": 266, "gfs_start": "July 02 2019 at 18", "gfs_end": "July 06 2019 at 00"}```

  ### gfslevels
  * Parameters: none
  * Returns: JSON response describing all the gfs forecast levels
  ```{"levels": ["atmosphere", "depthBelowLandLayer", "heightAboveGround", etc... ]}```

  ### timeseries
  * Parameters:<ul>
    * model **(required)**: (string) the name of the EO data model you want a timeseries for using the shortened name you get from the eodatamodels api call. Examples: 'gldas'
    * variable **(required)**: (string) the name of the variable you want data for using the shortened name you get from the appropriate api call for your model type (eg gldasvariables if you chose the gldas model). Examples: 'Tair_f_inst'
    * loc_type **(required)**: (string) description of the location you want data for, either Point, Polygon, or Shapefile.
    * coords **(required)**: (list of integers) for loc_type Point: a list containing first the longitude, from -180 to 180, then the latitude, from -90 to 90. Example [-100, 50]. For loc_type Polygon, an array of coordinates of the bounding box for your polygon in the longitude, latitude order as for points. The array should be 5 rows by 2 columns. Start with the bottom left point, list the points clockwise, and repeat the first point.
    * region <i>(dependant)</i>: (string) if loc_type is Shapefile, include the name of the region to get values for. Choices include (with capitalization and spaces) Africa, Asia, Australia, Central America, Europe, Middle East, North America, South America
    * time <i>(dependant)</i>: (string) if model is gldas, a string with the 4 digit year you're interesting in or else a string for 'alltimes'.
    * level <i>(dependant)</i>: (string) if model is gfs, a string with the forecast level to get data for.
    </ul>
  * Returns: JSON response describing the dates and files of available gfs data
  ```{"values": [etc...], etc...}```
