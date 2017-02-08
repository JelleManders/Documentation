# REST API: opvragen van subprofiel identifiers

Als je alleen maar de ID's van de subprofielen in een miniselectie wilt opvragen,
kan dat met een heel simpele methode. Je kunt een HTTP GET request sturen 
naar het volgende adres:

`https://api.copernica.com/view/$id/subprofileids?access_token=xxxx`

De code $id moet je vervangen door de nummerieke identifier van de 
miniselectie waar je de ID's van wilt opvragen.

## Beschikbare parameters

Deze methode ondersteunt geen parameters

## Geretourneerde velden

De methode retourneert een JSON array bestaande uit nummerieke identifiers.

## Voorbeeld in PHP

Het volgende PHP script demonstreert hoe je de API methode kunt aanroepen.

    // dependencies
    require_once('copernica_rest_api.php');
    
    // change this into your access token
    $api = new CopernicaRestApi("your-access-token");

    // do the call, and print result
    print_r($api->get("miniview/1234/subprofileids"));

Voor bovenstaand voorbeeld heb je de [CopernicaRestApi klasse](rest-php) nodig.
    

## Meer informatie

* [Overzicht van alle API calls](rest-api)
* [Subprofielen inclusief alle subprofieldata opvragen](rest-get-view-subprofiles)