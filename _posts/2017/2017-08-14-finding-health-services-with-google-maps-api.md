# Finding health services with Google Maps API

__14/08/2017__

![gmap](/assets/hank.jpg)

As part of a recent health [Hackathon](https://github.com/shusson/hank) we
wanted to collect some health information per suburb. Specifically we
were looking for things like "how many hospitals are in this suburb?" As
we were time constrained we decided to explore what the Google Maps API
could give us. We already knew that Google Maps could provide this
kind of information, e.g searching an area for hospitals. But could
the API provide similar results? And what kind of accuracy would we get?

Google provides a [few different libraries](https://developers.google.com/maps/web/)
across different languages. If you plan to do anything more than a
few simple queries I'd recommend you use one of the server side libraries
(google calls them [web services](https://developers.google.com/maps/web-services/client-library)).
The server side libraries provide nice features like `Automatic Rate Limiting`.

Before you can use any of the APIs you'll need to [get a key](https://developers.google.com/places/web-service/get-api-key).

The APIs are broken down into specific domains such as Directions or
Geolocation. For our use case we used the [Places API](https://developers.google.com/places/web-service/),
 which lets you search for places either by proximity or a text string.
 The [text string search](https://developers.google.com/places/web-service/search#TextSearchRequests)
 allows you to specify a radius but it will still return results outside
 the radius, i.e it uses the radius for ranking.
 In contrast the [proximity search](https://developers.google.com/places/web-service/search#PlaceSearchRequests)
 will only return places within a certain radius around a location.

Finding how many Hospitals are in Darlinghurst:

```javascript
const mc = require('@google/maps').createClient({
  key: 'YOUR_KEY'
});

const location = [-33.878101, 151.220386]; // Darlinghurst, AU
const r = 1000;
const request = {
    location: location,
    radius: r,
    type: 'hospital'
};

mc.placesNearby(request, function (err, res) {
    if (err) {
        console.log(err);
        return;
    }
    console.log(JSON.stringify(res.json.results.length));
    console.log(JSON.stringify(res.json.results.map(function (v) {
        return v.name;
    })));
});
```

If you run the above code it'll print out:

```bash
20
["Darlinghurst Medical Centre","Wordsworth House Medical Practice","Hyde Park Medical Centre - Sydney CBD","St Vincent's Private Hospital Sydney","Sydney Acupuncture Clinic","Sydney Day Surgery","Lord Surgery St Vincents's Private Hospital","Liverpool Street Medical Suites","Dr Maxine Szramka, Sydney Rheumatologist","Dr. Nick Vertzyas - Orthopaedic Surgeon","Ingrown Toenail Specialist Sydney","The Scottish Hospital","St. Vincents Hospital Alcohol & Drug Service","Tierney House","Blood & Marrow Transplant Network","Great Directions","Bourke Street Clinic","Centurion Healthcare","Potts Point Medical Practice","Creative Counsel"]
```

There's 2 obvious issues here:

  1. The API has capped the result at `20`.

  2. The result contains many places that I would not consider a hospital.

In regards to 1, the API let's you ask for another page of results using
 a `pagetoken` in the request, but will at most give you [60 results in
 total](https://developers.google.com/places/web-service/search#PlaceSearchPaging).

In regard to 2, we can improve our search by adding a `keyword` to the request:

```javascript
...
const request = {
    location: location,
    radius: r,
    type: 'hospital',
    keyword: 'hospital`
};
...
```

Which will result in:

```bash
9
["St Vincent's Hospital Sydney","St Vincent's Private Hospital Sydney","St. Vincents Hospital Alcohol & Drug Service","East Sydney Private Hospital","The Scottish Hospital","Hospital","Lord Surgery St Vincents's Private Hospital","Tierney House","Level 4, Xavier Building"]
```

So the Places API provides some filters that let you narrow down your results,
but the accuracy isn't great. The Places API also caps results at 60, which
was ok for the `hospital` query but would probably be limiting for queries
like `doctors` or `clinics`. For the Hackathon, the Places API was
sufficient at getting an overview of health services in a suburb, but further
investigation is required if we wanted to expand the idea further.

Sample source code available on [github](https://github.com/shusson/hank/blob/master/server/find-places.js)
You'll have to use your own key, as the one that's there has been deleted.
