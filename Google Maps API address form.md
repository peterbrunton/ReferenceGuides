#Google Places API address form

Use Google Places API to autocomplete shipping address form.

##Goal

Use the [Google Maps Javascript API](https://developers.google.com/maps/documentation/javascript/examples/places-autocomplete-addressform) to let customers autocomplete their shipping address information.

Instead of relying on the browser's autofill settings, a dropdown menu powered by Google will present options and fill out the rest of the form to the best of its ability.

![http://g.recordit.co/OS8l96ULO8.gif](http://g.recordit.co/OS8l96ULO8.gif)

##Steps

### Step 1 - Call the needed external scripts

In the `<head>`, call three external scripts:

1. A recent version of jQuery
2. The [Google Places Javascript API](https://developers.google.com/maps/documentation/javascript/examples/places-autocomplete-addressform)
3. Our own customized **addressAutocomplete.js.liquid** file below that we'll add to the Assets directory.

*Note*: The addressAutocomplete.js.liquid file has been customized for **checkout_v2**.

```html
  {{ '//code.jquery.com/jquery-1.12.0.min.js' | script_tag }}
  {{ 'https://maps.googleapis.com/maps/api/js?v=3.exp&signed_in=true&libraries=places' | script_tag }}
  {{ 'addressAutocomplete.js' | asset_url | script_tag }}
```

### Step 2 - addressAutocomplete

Add the following script to an asset named `addressAutocomplete.js.liquid`.

```javascript
addressAutocomplete = {};

var autocomplete;
var addressType;
var i;
var val;
var componentForm = {
  route: {
    name: 'checkout[shipping_address][address1]',
    format: 'long_name'
  },
  locality: {
    name: 'checkout[shipping_address][city]',
    format: 'long_name'
  },
  country: {
    name: 'checkout[shipping_address][country]',
    format: 'long_name'
  },
  administrative_area_level_1: {
    name: 'checkout[shipping_address][province]',
    format: 'long_name'
  },
  postal_code: {
    name: 'checkout[shipping_address][zip]',
    format: 'short_name'
  }
};

addressAutocomplete.mapInitialize = function() {
  // Create the autocomplete object, restricting the search to geographical location types.
  autocomplete = new google.maps.places.Autocomplete(
    /** @type {HTMLInputElement} */
    (document.getElementsByName(componentForm.route.name)[0]), {
      types: ['geocode']
    });
  // When the user selects an address from the dropdown, populate the address fields in the form.
  google.maps.event.addListener(autocomplete, 'place_changed', function() {
    fillInAddress();
  });
};

// [START region_fillform]
function fillInAddress() {
  // Get the place details from the autocomplete object.
  var place = autocomplete.getPlace();
  var placeComponents = place.address_components;

  for (var component in componentForm) {
    if (componentForm.hasOwnProperty(component)) {
      document.getElementsByName(componentForm[component].name)[0].value = '';
      document.getElementsByName(componentForm[component].name)[0].disabled = false;
    }
  }

  // Trigger country select change first to reveal correct province selector
  for (i = 0; i < placeComponents.length; i++) {
    addressType = placeComponents[i].types[0];

    if (componentForm[addressType]) {
      val = place.address_components[i][componentForm[addressType].format];

      if (componentForm[addressType].name == componentForm.country.name) {
        document.getElementsByName(componentForm[addressType].name)[0].value = val;
        Checkout.$('[name="' + componentForm.country.name + '"]').trigger('change');
      }
    }
  }

  // Get each component of the address from the place details and fill the corresponding field on the form.
  for (i = 0; i < placeComponents.length; i++) {
    addressType = placeComponents[i].types[0];

    if (componentForm[addressType]) {
      val = place.address_components[i][componentForm[addressType].format];

      if (addressType == 'route' && placeComponents[0].types[0] == "street_number") {
        var street_number = placeComponents[0].long_name;
        document.getElementsByName(componentForm.route.name)[0].value = street_number + ' ' + val;
      } else if (componentForm[addressType].name !== componentForm.country.name) {
        document.getElementsByName(componentForm[addressType].name)[0].value = val;
      }

    }
  }
}
// [END region_fillform]

{% comment %} 
  Comment out the code below to stop the browser notification
  stating that this site wants permission to use your location.
{% endcomment %}

// [START region_geolocation]
// Bias the autocomplete object to the user's geographical location, as supplied by the browser's 'navigator.geolocation' object.
function geolocate() {
  if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition(function(position) {
      var geolocation = new google.maps.LatLng(
        position.coords.latitude, position.coords.longitude);
      var circle = new google.maps.Circle({
        center: geolocation,
        radius: position.coords.accuracy
      });
      autocomplete.setBounds(circle.getBounds());
    });
  }
}
// [END region_geolocation]
```


### Step 3 - Call the addressAutocomplete function

Use the `Shopify.Checkout.step` function to determine what step of the checkout we are on. When we are
are the **contact_information** step, we'll call the `addressAutocomplete.mapInitialize()`function.

The keydown event script below makes it possible for users to press 'Enter' when selecting
an address without automatically submitting the form.

*Note*: If you've decided to not use the `geolocate()` function, you'll want to comment out
the line referring to it in this code block.

```javascript
(function($) {
  'use strict';

  $(document).on('page:load page:change', function() {

    if (Shopify.Checkout.step === 'contact_information') {
      addressAutocomplete.mapInitialize();

        var $address = $('[name="checkout[shipping_address][address1]"]');

        $address.attr('onFocus', geolocate());

        $address.on('keydown', function(e) {
        var x = e.which;
        
        if (x === 13) {
          e.preventDefault();
        }
      });
    }
  });
})(jQuery);
```

## Browser notifications

The `geolocate()` function biases address results to the customer's IP address' geographical location. While helpful, it requires the user to give his or her explicit permission to let the browser use their location. This comes in the form of a notification like the screenshot below. If you think this is a pain in the butt, then just comment out the code and don't use it.

![http://take.ms/s1o0M](http://take.ms/s1o0M)

## Considerations

Note that we are looping through the `placeComponents` object array twice. The first loop is targeted only at the Country selector, and the second loop targets the rest. This is because the object array that google gives us doesn't provide any unique id's to target specific information, and in order to make sure the correct provinces are returned in the checkout select menu, we must ensure that we update the Country selector first. If the province came before the country in the object array this wouldn't be an issue, but alas that is not the case.

Ideally, it would be nice to update the order of the object array given to us before we loop through it but at this point in time I can't think of a way to do it.

If anyone can find a better approach to this remedy this issue please update this reference guide.
