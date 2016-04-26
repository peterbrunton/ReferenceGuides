# Test Inputs with Regex

![http://g.recordit.co/McS1c09WJd.gif](http://g.recordit.co/McS1c09WJd.gif)

## The Issue
Let's say you are a merchant selling your wares in foreign countries, however, the delivery company you use can only read english, so any orders you receive with their address written in a non-english language can not be delivered. If this sounds familiar, then you've probably encountered [Mantraband](https://github.com/Shopify/plus-theme-support/issues/315).

The problem is, you can't reliably prevent the user from entering their address in non-latin characters using keycodes because each browser seems to have it's own idea of what the keycode should be.

## The Fix
Now that we've established that using keycodes won't work, we're going to have to let the user enter their address in non-latin characters, but when they attempt to continue to the next step, we search the customer information input fields for specific characters, throw an alert and add the error helper classes to the relevant input fields if we find any. This checks every input field on every step of checkout.

## The Code

Firstly, lets set up a theme setting for the error message which will be displayed if the characters in input fields are invalid.
```json
{
    "name": "Test Inputs with Regex [Plus]",
    "settings": [
      {
        "type": "header",
        "content": "Limit input with regex"
      },
      {
        "type": "checkbox",
        "id": "plus_enable_restrict_w_regex",
        "label": "Enable restrict input with regex",
        "default": true,
        "info": "Enabling this will retrict input with the regex expression"
      },
      {
        "type": "textarea",
        "id": "input_invalid_characters",
        "label": "Error message to display"
      }
    ]
  }
```

Next, lets either put this at the bottom of the `checkout.liquid` file, or better yet, include this in a `js` snippet, eg. `checkout-input-validation.js.liquid` and include that inside of `checkout.liquid`.

**This requires jQuery**  
`{{ '//ajax.googleapis.com/ajax/libs/jquery/1.12.0/jquery.min.js' | script_tag }}`

```javascript
{% if plus_enable_restrict_w_regex %}

<style>
.custom-error-message{
  display:block;  
  line-height: 1.3em;
  font-size: 12px;
  margin: 0.75em 0 0.25em;
  color: #b564ff;
}
</style>

<script>
  
(function($) {
  
  $(document).on('ready', function() {
    
    if (Shopify.Checkout.step === 'contact_information') {
      
      var regex = /\b[p](ost(al)?)?\.?\s?([o](ffice)?)?\.?\s?([b](ox))?(\s+)?[\d]+\b/i;
      var isValid = true;
      //var fieldErrorClass = 'field--error';
      //var fieldErrorMessageSelector = '.field__message--error';
      var errorText = {{ settings.input_invalid_characters | json }};
      var $inputs = $("input[name='checkout[shipping_address][address1]']");
      $inputs.off('blur.addressInput');
          
      var regexCheckFn = function(elem) {
        
        var $current = $(elem);
        
        if (regex.test($current.val())) { 
          
          var errorMessage = "<p class='custom-error-message'>"+ errorText +"</p>";        
          isValid = false;
          if ($('p.custom-error-message').length == 0) {
            $current.after(errorMessage);
          }
        
        } else {
                    
          isValid = true;
          
          if ($('p.custom-error-message').length == 1) {
            $('p.custom-error-message').remove();  
          }
          
        }
      };// regexCheckFn
      
      // Call regex check on form submit
      Checkout.$(document).on('submit', '[data-step] form', function(e) {
        
        e.preventDefault();
        e.stopPropagation();
      
        $inputs.each(function() {
          regexCheckFn(this);
        });
      
        if (!isValid) {
          return false;
        } else {
          $(this).submit();
        }
      });
      
      // Call regex check on blur
      $inputs.on('blur.addressInput', function() {
        isValid = true; //always start true
        regexCheckFn(this);
      });
    
   	}// step equals contact info.
  
  });// on ready()
  
})(jQuery);
</script>
 
{% endif %}

```

### REGEX Expressions

Here are some commonly used regex expressions to aid in your testing. If you'd like to test these out, use [regexr](http://www.regexr.com/).

**Integer & Decimal Numbers**  
`/(?:\d*\.)?\d+/g`

**Integer, Hyphen, Space, & Dots**  
`/^[0-9-.\s]+$/g`

**Words**  
`/[a-zA-Z]+/g`

**Digits**  
`/^[0-9]+$/g`

**PO Box Address**  
`/\b[p](ost(al)?)?\.?\s?([o](ffice)?)?\.?\s?(box)?(\s+)?[\d]+\b/gmi`
