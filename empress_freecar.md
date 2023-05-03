# carquest
These are the steps to find nearby cars that are available for the next 12 hours.

Modo makes it very easy to query their cars' details. They got [nice docs](https://bookit.modo.coop/api/v2#car_list) for their API, but I tried to get the gist of what you need to do here. No need to authenticate, but remember this is only for getting the information. Just reading the information from the servers, you need to use the app in the end to make a booking. I am not detailing the endpoints here, please see their docs for futher info.

These are the endpoints (JSON / GET or POST) 

|  name |  ednpoint |  function |
|---|---|---|
|  Car List |  	https://bookit.modo.coop/api/v2/car_list |  all cars' details including base carpark |
|  Location List | 	https://bookit.modo.coop/api/v2/location_list  |  carparks and their details  |
|  Nearby | 	https://bookit.modo.coop/api/v2/nearby  | list nearby carparks (no car info) |

## routine

I used Fetch API to sketch out the routine. Use Chrome with Developer Tools to open `modo_monday.html` - this way you can see the console messages (image at end of this page). I just used urls with Fetch API, which means method is GET. The code is incomplete, I have not managed to list the cars in the choosen carkpark or determine whether they are free - since that would come after it. I think it is a JSON data formatting issue, I am stuck there. Still, I think I have a good idea of what should be done. 

### (1) Check which carparks are nearby. 
Each car is assigned to a specific parking spot/base where it can be rented from and must be returned to. Use the `latitude` and `logitude` values of your hotel and I suggest using the `distance` parameter with 500 meters. If you think there are too many finds, you can decrease this distance but the chance of finding an available car becomes smaller.
```
https://bookit.modo.coop/api/v2/nearby?lat=48.42139&long=-123.36723&distance=500
```
You get a number of 3-digit `LocationID`s that are nearby, I got six. 
`Neighbourhood` is a parameter on the carparks (`LocationID`s), so you could also list carparks in your own neighbourhood (Downtown) or adjecent to it (Harris Green, for example), but using the coordinates with distance is probably a more efficient move. 

The result that comes back is a *ranked list*, so the first one is the closest and the last one is the most far away carpark.  

An example `LocationID` (carpark):

```
"Locations": {
      "802": {
        "ID": "802",
        "Name": "Rupert Terrace",
        "ShortDescription": "Victoria - Rupert Terrace & Quadra Street",
        "Region": "Greater Victoria",
        "City": "Victoria",
        "Neighbourhood": "Downtown Victoria",
        "Latitude": "48.420932",
        "Longitude": "-123.361846",
        "ExceptionClass": null
      }
```
### (2) Query all the modo cars and narrow down the results to the chosen carpark. 
This give you a gigantic list of Modo's active cars with all their details.
```
https://bookit.modo.coop/api/v2/car_list
```
This is what one car entry looks like:
```
"600": {
        "ID": "600",
        "Make": "Fiat",
        "Model": "500",
        "Year": "2015",
        "Colour": "Red",
        "Identifier": null,
        "Category": "2-door Hatchback",
        "Class": "Car",
        "ExceptionClass": null,
        "Seats": "4",
        "Accessories": [
          "audio: aux input",
          "audio: bluetooth",
          "audio: mp3 cd player",
          "audio: usb audio",
          "cruise control"
        ],
        "Location": [
          {
            "LocationID": "383",
            "StartTime": null,
            "EndTime": null
          }
        ]
      }
```
Use the `LocationID` obtained in the first step to and filter to the cars that are currenly in that choosen location. 
### (3) Check if there is an available car in the closest carpark.
You found which cars are in the closest carpark, now send a query specifically for each one to see if it is available:
```
https://bookit.modo.coop/api/v2/car_list?car_id=28
```
Use `StartTime` and `EndTime` (convert to Epoch timestamps) to see if a car is available for 12 hours (between now and +44000 seconds). 44k secs is a little more than 12 hours, but making a choice and getting to the car also takes time. Cars can be booked well into the future, if a car has booked journeys it may not be relevant for you. If there are bookings on the car, the closest booking in time needs to have a `StartTime` of now plus 44000 secs in Epoch to be suitable for you. This is what a booked journey looks like:
```
"Location":[
{"LocationID":"734","StartTime":"1683896400","EndTime":"1683932400"},
{"LocationID":"734","StartTime":"1683205200","EndTime":"1683241200"}]
```
Nothing booked, free car:
```
"Location":[{"LocationID":"602","StartTime":null,"EndTime":null}]
```
Note that **both** need to be null. You timzone in Victoria is America/Vancouver (PDT) and the offset (difference to Greenwich Time/GMT) is -07:00 or in seconds -25200.

If there are no available cars in the nearest carpark, start checking the second nearest one and proceed down that list untill you find one. Or theoretically there could be no cars and then you need to enlarge your search radius.

### (4) Show the properties for the closest carpark with an available car to the user.
Get the carpark's details to display the in the browser/app window.
```
https://bookit.modo.coop/api/v2/location_list?location_id=431
```
### (5) Show the properties of the available cars (make, color) in the closest carpark to the user.
Get the availabel cars' details to display the in the browser/app window. Make sure you show `Make` `Model` and `Colour`, so the car is easy to find.
```
https://bookit.modo.coop/api/v2/car_list?car_id=961
```
I assumed there would be multiple finds/cars that you can choose from, so you may also list accessories of the availble cars to help you choose.

- new coordinates if the ride is not from the hotle, but lets say maybe your fav breakfast/hangout place or somewhere else.

### stuck: my JSON/filter problem

I have a list of all cars but cannot narrow it down. Think what I got is a complex nested JSON object. Keys are the car `ID`s and the value is all the car data including the same car `ID` again. In this format, I could not manage to filter it (error is similar to ?uncaught promise not a function?). If I managed to remove the key element from all cars, and just have an array of cardata without the initial CarID as the key, think the filter would work. Who know, maybe it is another problem, but I am not getting there yet. I am not displaying any finding on the html page because it is not entirely sure that there is a free car in the carpark, checking needs to be done.

### Chrome and devtools for console log
![Chrome and devtools](chrome_devtool.png "Chrome and devtools")
