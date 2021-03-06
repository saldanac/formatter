# Formatter
[![Build Status](https://travis-ci.org/collab-corp/formatter.svg?branch=master)](https://travis-ci.org/collab-corp/formatter)
[![StyleCI](https://styleci.io/repos/119897298/shield?branch=master)](https://styleci.io/repos/119897298)

A Laravel formatting utility package.

We often find ourselves converting request input to meet certain formats. We do it before validation, maybe after validation, or we do it in our mutators. This package was designed to ease that process and to keep our controller/model code cleaner. This package mostly uses laravel's helper methods on top of its own custom methods, there is also some formatting using the Carbon date library.

## Installation

`composer require collab-corp/formatter`

## Binding / Package Discovery

By default this package is auto discovered in Laravel 5.5 with its service provider automatically binding a Formatter using the `collab-corp.formatter` key binding. You can resolve a binding as follows:

```php

$formatter = app('collab-corp.formatter'); //returns Formatter instance with a default null value

$formatter = $formatter->setValue('foobar');

```

For lower versions of laravel you will have to manually register `\CollabCorp\Formatter\FormatterServiceProvider::class`
in your `/config/app.php` file.



## Requires

This package makes use of:

* bcmath extension


## Basic Use:
You can new up a formatter
`new Formatter('yourValue')->{someMethod}()->get();`

In addition to calling get, it is also possible to get the results by casting the formatter to a string `(string)Formatter::create('something')->{method}();`


or call the static method create `Formatter::create('yourValue')->{someMethod}()->get();`

Here are a few examples using laravel's mutators :

```php

<?php

use CollabCorp\Formatter\Formatter;
//etc.
class SomeModel extends Model{

    public function getProductPriceAttribute(){
        //format our number to 2 decimal places or however many places you want
        return new Formatter($this->attributes['price'])->decimals(2)->get();

        //or can call static create
        return Formatter::create($this->attributes['price'])->decimals(2)->get();
    }

    public function getPhoneNumberFormattedAttribute(){
        //format a 10 or 11 digit number to be in parenthesis US format:  x(xxx)xxx-xxxx
        return new Formatter($this->attributes['phone_number'])->phone()->get();

    }

}

````

## Method Chaining

These above examples are nothing special, you could do the same  with a simple method/function call yourself. The real usefulness of this package is the ability to method chain and process many conversions on a input:

```php
<?php
    //convert our phone number to a parenthesis format but first make sure to strip any non numeric characters off first
    $result = new Formatter($request->phone_number)
                        ->onlyNumbers()
                        ->phone()
                        ->get();

   //another example
   $result = new Formatter(2)
                       ->add(2)
                       ->multiply(3)
                       ->finish("%")
                       ->get(); // 12%


```

This makes it convienient to run multiple conversions/formatting for our value. It also makes it easier to keep code clean and having to combine multiple methods yourself. Take the above second example. Doing the method calls manually would look something like this:

```php

    Str::finish(bcmul(bcadd(2, 2), 3), "%");

```

Yuck! right? Although this was a rather small example and not bad, imagine if you had to do 3 other conversions? Keep your code cleaner and more readable with method chaining :D

## Mass Conversions

Another useful and probably the best reason for use of this package is processing multiple formatters on a given array of input such as the reques input or model attributes.The workflow for this is very similiar to laravel validation, so it should feel very natural to you. Here's an example of having middleware automatically format/convert input before the request hits the controller:


```php

 <?php

    public function handle($request, Closure $next, $guard = null)
    {

          $formatters=[
              //convert to parenthesis phone format after stripping all non numeric characters
              'phone'=>'onlyNumbers|phone',
              //format to 2 decimal places
              'price'=>'decimals:2',
               // trim off % signs,convert to decimal percent with 2 decimal places.
              'tax'=>'trim:%|percentage:2',
              // make this input slug friendly
              'page'=>'slug'

          ];

          //returns collection of new input converted.
          $convertedInput = Formatter::convert($formatters, $request->all());

          //replace existing request data with new converted data,
          $request->replace($convertedInput->all());

          return $next($request);
    }


```
Very similiar to laravel validation right? When defining the formatters,use the name of the input in the request as the key and the value being the formatter(s) you want to run on that input. Seperate each method with a pipe character `|`.
If the formatter method you want to call needs parameters then specify paramter input with a colon `:` then pass the parameters in a comma delimited list in the order that the method accepts them, refer to the methods  section to see what <a href="#methods">methods</a> require/accept parameters.

`ex: add:2,2`

You could also define a method on your models and pass in `$this->attributes`  vs defining a mutator for each attribute.

# Methods are whitelisted

By default all formatter methods are checked against a whitelist at run time. This is just a precautionary measure to avoid allowing client side requests to make calls to formatter class methods. Example:Consider a UI that allows clients to determine formatter methods. The same goes for macro added methods. If the method is not in the whitelist or was not added via the macro trait, then naturally they are considered undefined methods.

## Patterns
The above example is being explicit in its request keys, but you could also specify pattern input keys using asterisk to  match input keys and process them if they match the pattern:

```php
<?php

    $formatters=[
        '*phone'=>'onlyNumbers|phone', //run methods on any input that has a name that ends with phone
        'phone*'=>'onlyNumbers|phone', //run methods on any input that has a name that starts with phone
        '*phone*'=>'onlyNumbers|phone', //run methods on any input that has a name that has the word phone in it.
    ];

    $convertedInput = Formatter::convert($formatters, $request->all());

```

<strong>Note: Pattern matching with nested associated arrays is not supported. You must be explicit with nested associative arrays formatters:</strong>


```php


$formatters=[

   'applicant.*name*'=>'titleCase' // this is not supported
   'applicant.name'=>'titleCase' //this is. Were explicitly telling the formatter to format $request->input('applicant.name');

];

```

## Macroable

The `Formatter` class is macroable which means you can add extra formatting methods on run time. Heres an example:

```php
Formatter::macro('toUpper', function () {
    return $this->setValue(strtoupper($this->value));
});

$upper= (new Formatter("sergio"))->toUpper()->get(); // "SERGIO"

```


##  Methods

####  String Formatters:

<ul>
    <li>
        <a href="#ssn">ssn</a>
    </li>
    <li>
        <a href="#phone">phone</a>
    </li>
    <li>
        <a href="#truncate">truncate</a>
    </li>
    <li>
        <a href="#finish">finish</a>
    </li>
    <li>
        <a href="#start">start</a>
    </li>
    <li>
        <a href="#prefix">prefix</a>
    </li>
    <li>
        <a href="#suffix">suffix</a>
    </li>
    <li>
        <a href="#insertevery">insertEvery</a>
    </li>
    <li>
        <a href="#camelcase">camelCase</a>
    </li>
    <li>
        <a href="#snakecase">snakeCase</a>
    </li>
    <li>
        <a href="#titlecase">titleCase</a>
    </li>
    <li>
        <a href="#kebabcase">kebabCase</a>
    </li>
    <li>
        <a href="#studlycase">studlyCase</a>
    </li>
    <li>
        <a href="#slug">slug</a>
    </li>
    <li>
        <a href="#plural">plural</a>
    </li>
    <li>
        <a href="#limit">limit</a>
    </li>
    <li>
        <a href="#encrypt">encrypt</a>
    </li>
    <li>
        <a href="#decrypt">decrypt</a>
    </li>
    <li>
        <a href="#bcrypt">bcrypt</a>
    </li>
    <li>
        <a href="#replace">replace</a>
    </li>
    <li>
        <a href="#onlyalphanumeric">onlyAlphaNumeric</a>
    </li>
    <li>
        <a href="#onlynumbers">onlyNumbers</a>
    </li>
    <li>
        <a href="#onlyletters">onlyLetters</a>
    </li>
    <li>
        <a href="#trim">trim</a>
    </li>
    <li>
        <a href="#ltrim">ltrim</a>
    </li>
    <li>
        <a href="#rtrim">rtrim</a>
    </li>
    <li>
        <a href="#url">url</a>
    </li>
</ul>

### Math Formatters

<ul>
    <li>
        <a href="#decimals">decimals</a>
    </li>
    <li>
        <a href="#add">add</a>
    </li>
    <li>
        <a href="#subtract">subtract</a>
    </li>
    <li>
        <a href="#multiply">multiply</a>
    </li>
    <li>
        <a href="#divide">divide</a>
    </li>
    <li>
        <a href="#power">power</a>
    </li>
    <li>
        <a href="#percentage">percentage</a>
    </li>
</ul>

### Date Formatters
Note: These simply are methods called using the Carbon Library. These are the only available methods on the Formatter:
<ul>
    <li>
        <a href="#tocarbon">toCarbon</a>
    </li>
    <li>
        <a href="#format">format</a>
    </li>
    <li>
        <a href="#settimezone">setTimezone</a>
    </li>
    <li>
        <a href="#addyears">addYears</a>
    </li>
    <li>
        <a href="#addmonths">addMonths</a>
    </li>
    <li>
        <a href="#addweeks">addWeeks</a>
    </li>
    <li>
        <a href="#adddays">addDays</a>
    </li>
    <li>
        <a href="#addhours">addHours</a>
    </li>
    <li>
        <a href="#addminutes">addMinutes</a>
    </li>
    <li>
        <a href="#addseconds">addSeconds</a>
    </li>
    <li>
        <a href="#subyears">subYears</a>
    </li>
    <li>
        <a href="#submonths">subMonths</a>
    </li>
    <li>
        <a href="#subweeks">subWeeks</a>
    </li>
    <li>
        <a href="#subdays">subDays</a>
    </li>
    <li>
        <a href="#subhours">subHours</a>
    </li>
    <li>
        <a href="#subminutes">subMinutes</a>
    </li>
    <li>
        <a href="#subseconds">subSeconds</a>
    </li>
</ul>

* ### ssn
    convert a 9 numeric value to a social security format:
    ```php
    //'123-45-6789'
    Formatter::create('123456789')->ssn()->get();
    ```

 * ### phone
    convert a 10 or 11 numeric value to a parenthesis format:
    ```php
    //'(123)456-7890'
    Formatter::create('1234567890')->phone()->get();
    //'1(123)456-7890'
    Formatter::create('11234567890')->phone()->get();
* ### truncate
    truncate off the specified number of characters of the value:
    ```php
    //'foo'
    Formatter::create('foobar')->truncate(3)->get();
    ```
* ### finish
    add the specified character to the end of the string if it doesnt already contain it:
    ```php
    //'foobar'
    Formatter::create('foo')->finish('bar')->get();
    //'foobar'
    Formatter::create('foobar')->finish('bar')->get();
    ```

* ### start
    add the specified character to the start of the string if it doesnt already contain it:
    ```php
    //'foobar'
    Formatter::create('foobar')->start('foo')->get();
    //'foobar'
    Formatter::create('bar')->start('foo')->get();
    ```

 * ### prefix
    add the specified character to the start of the string:
    ```php
    //'foofoobar'
    Formatter::create('foobar')->prefix('foo')->get();
    ```
  * ### suffix
    add the specified character to the end of the string:
    ```php
    // 'foobarfoo'
    Formatter::create('foobar')->suffix('foo')->get();
    ```
  * ### insertEvery
    add the specified character every nth characters:
    ```php
    // '1234 5678 9012  3456'
    Formatter::create('1234567890123456')->insertEvery(4, " ")->get();
    ```
  * ### camelCase
    convert value to camel case:
    ```php
    //'fooBar'
    Formatter::create('foo bar')->camelCase()->get();
    ```
  * ### snakeCase
    convert value to snake case:
    ```php
    //'foo_bar'
    Formatter::create('foo bar')->snakeCase()->get();
    ```
  * ### titleCase
    convert value to snake case:
    ```php
    //'Foo Bar'
    Formatter::create('foo bar')->titleCase()->get();
    ```
  * ### kebabCase
    convert value to kebab case:
    ```php
    // 'foo-bar'
    Formatter::create('foo bar')->kebabCase()->get();
    ```
  * ### studlyCase
    convert value to kebab case:
    ```php
    //'FooBar'
    Formatter::create('foo bar')->studlyCase()->get();
    ```
  * ### slug
    convert value to a slug friendly string:
    ```php
    //'foo-bar'
    Formatter::create('foo bar')->slug()->get();
    ```
  * ### plural
    convert value to its plural form:
    ```php
    //'children'
    Formatter::create('child')->plural()->get();
    ```
  * ### limit
    limit the string the first n of characters:
    ```php
    //'child'
    Formatter::create('children')->limit(5)->get();
    ```
  * ### encrypt
    encrypt the value:
    ```php
    //'{some encrypted string}'
    Formatter::create('someString')->encrypt()->get();
    ```
  * ### decrypt
    decrypt the value:
    ```php
    //the original string
    Formatter::create('{some encrypted string}')->decrypt()->get();
    ```
  * ### bcrypt
    hash the value with bcrypt:
    ```php
    //'{some hashed string result}'
    Formatter::create('secret1')->bcrypt()->get();
    ```
  * ### replace
    replace the given character with the given replacement character, defaults to empty string for replacement:
    ```php
    //'bar'
    Formatter::create('foobar')->replace('foo')->get();
    //'poobar'
    Formatter::create('foobar')->replace('foo', 'poo')->get();
    ```
  * ### onlyAlphaNumeric
    replace non alphanumeric characters, including spaces, unless specified by 1st parameter:
    ```php
    //'foobar123test'
    Formatter::create('foobar123 &$*&$(#(*test')->onlyAlphaNumeric()->get();
    //'foobar123 test 123'
    Formatter::create('foobar123 &$*&$(#(*test 123',true)->onlyAlphaNumeric()->get();
    ```
  * ### onlyNumbers
    removes all characters that are not numbers from the value:
    ```php
    //'123'
    Formatter::create('sfsdfs123')->onlyNumbers()->get();
    ```
  * ### onlyLetters
    removes all characters that are not letters from the value:
    ```php
    //'test'
    Formatter::create('#(@)!@test123')->onlyLetters()->get();
    ```
  * ### trim
    removes all leading and ending spaces/characters from the value:
    ```php
    //'something'
    Formatter::create('   something     ')->trim()->get();
    //something
    Formatter::create('####something####')->trim("#")->get();
    ```
  * ### ltrim
    removes all leading spaces/characters from the value:
    ```php
    //'something'
    Formatter::create('   something')->trim()->get();
    //something####
    Formatter::create('####something####')->trim("#")->get();
    ```
  * ### rtrim
    removes all ending spaces/characters from the value:
    ```php
    //'    something'
    Formatter::create('    something    ')->rtrim()->get();
    //####something
    Formatter::create('####something####')->rtrim("#")->get();
    ```

  * ### url
    removes all ending spaces/characters from the value:
    ```php
    //'http://{APP_URL}/something'
    Formatter::create('something')->url()->get();
    ```

    ##

  * ### decimals
    format a number to have the speficied number of decimal places:
    ```php
    //20.00
    Formatter::create(20)->decimals(2)->get();
    ```
  * ### add
    add a given number to the current numeric value.Automatically scales 0 decimal places unless specified as 2nd param:
    ```php
    //22
    Formatter::create(20)->add(2)->get();
    //22.00
    Formatter::create(20)->add(2,2)->get();
    ```
  * ### subtract
    subtract a given number from the current numeric value. Automatically scales 0 decimal places unless specified as 2nd param:
    ```php
    //18
    Formatter::create(20)->subtract(2)->get();
    //18.00
    Formatter::create(20)->subtract(2,2)->get();
    ```
  * ### divide
    divide a the current numerica value by a given number. Automatically scales 0 decimal places unless specified as 2nd param:
    ```php
    //18
    Formatter::create(20)->divide(2)->get();
    //18.00
    Formatter::create(20)->divide(2,2)->get();
    ```
 * ### power
    raise the current value by a given power. Automatically scales 0 decimal places unless specified as 2nd param:
    ```php
    //400
    Formatter::create(20)->power(2)->get();
    //400.00
    Formatter::create(20)->power(2,2)->get();
    ```
 * ### percentage
    convert the value to a decimal percentage. Automatically scales 2 decimal places unless specified as 2nd param:
    ```php
    //0.20
    Formatter::create(20)->percentage()->get();
    //0.200
    Formatter::create(20)->percentage(2,2)->get();
    ```

  ##

* ### toCarbon
    convert the value to a Carbon\Carbon instance:
    ```php
    //Carbon instance "2030-12-22 00:00:00"
    Formatter::create("12/22/2030")->toCarbon()->get();
    ```
* ### format
    convert the Carbon\Carbon instance to a specified date/time string:
    ```php
    $formatter = Formatter::create("12/22/2030")->toCarbon();
    //"December 22, 2030 00:00:00"
    $formatter->format('F d, Y')->get();
    ```

* ### setTimezone
    set the specified timezone on the Carbon instance:
    ```php
    $formatter = Formatter::create("12/22/2030")->toCarbon();
    //Carbon instance with given timezone set "2030-12-22 00:00:00".
    $formatter->setTimezone('America/Toronto')->get();
    ```
* ### addYears
    add the given number of years to  the Carbon instance:
    ```php
    $formatter = Formatter::create("12/22/2030")->toCarbon();
    //Carbon instance "2032-12-22 00:00:00".
    $formatter->addYears(2)->get();
    ```
 * ### addMonths
    add the given number of months to  the Carbon instance:
    ```php
    $formatter = Formatter::create("12/22/2030")->toCarbon();
    //Carbon instance "2031-02-22 00:00:00".
    $formatter->addMonths(2)->get();
    ```
  * ### addWeeks
    add the given number of weeks to  the Carbon instance:
    ```php
    $formatter = Formatter::create("12/22/2030")->toCarbon();
    //Carbon instance "2031-01-04 00:00:00".
    $formatter->addWeeks(2)->get();
    ```
  * ### addDays
    add the given number of days to  the Carbon instance:
    ```php
    $formatter = Formatter::create("12/22/2030")->toCarbon();
    //Carbon instance "2030-12-24 00:00:00".
    $formatter->addDays(2)->get();
    ```
  * ### addHours
    add the given number of hours to  the Carbon instance:
    ```php
    $formatter = Formatter::create("2030-12-22 02:02:02")->toCarbon();
    //Carbon instance "2030-12-22 04:02:02".
    $formatter->addHours(2)->get();
    ```
  * ### addMinutes
    add the given number of minutes to  the Carbon instance:
    ```php
    $formatter = Formatter::create("2030-12-22 02:02:02")->toCarbon();
    //Carbon instance "2030-12-22 02:04:02".
    $formatter->addMinutes(2)->get();
    ```
  * ### addSeconds
    add the given number of seconds to  the Carbon instance:
    ```php
    $formatter = Formatter::create("2030-12-22 02:02:02")->toCarbon();
    //Carbon instance "2030-12-22 02:02:04".
    $formatter->addSeconds(2)->get();
    ```
  * ### subYears
    sub the given number of years to  the Carbon instance:
    ```php
    $formatter = Formatter::create("12/22/2030")->toCarbon();
    //Carbon instance "2028-12-22 00:00:00".
    $formatter->subYears(2)->get();
    ```
 * ### subMonths
    subtract the given number of months to  the Carbon instance:
    ```php
    $formatter = Formatter::create("12/22/2030")->toCarbon();
    //Carbon instance "2030-10-22 00:00:00".
    $formatter->subMonths(2)->get();
    ```
  * ### subWeeks
    subtract the given number of weeks to  the Carbon instance:
    ```php
    $formatter = Formatter::create("12/22/2030")->toCarbon();
    //Carbon instance "2030-12-08 00:00:00".
    $formatter->subWeeks(2)->get();
    ```
  * ### subDays
    subtract the given number of days to  the Carbon instance:
    ```php
    $formatter = Formatter::create("12/22/2030")->toCarbon();
    //Carbon instance "2030-12-20 00:00:00".
    $formatter->subDays(2)->get();
    ```
  * ### subHours
    subtract the given number of hours to  the Carbon instance:
    ```php
    $formatter = Formatter::create("2030-12-22 02:02:02")->toCarbon();
    //Carbon instance "2030-12-22 00:02:02".
    $formatter->subHours(2)->get();
    ```
  * ### subMinutes
    subtract the given number of minutes to  the Carbon instance:
    ```php
    $formatter = Formatter::create("2030-12-22 02:02:02")->toCarbon();
    //Carbon instance "2030-12-22 02:00:02".
    $formatter->subMinutes(2)->get();
    ```
  * ### subSeconds
    subtract the given number of seconds to  the Carbon instance:
    ```php
    $formatter = Formatter::create("2030-12-22 02:02:02")->toCarbon();
    //Carbon instance "2030-12-22 02:02:00".
    $formatter->subSeconds(2)->get();
    ```


## Contribute

Contributions are always welcome in the following manner:
- Issue Tracker
- Pull Requests
- Collab Corp Slack(Will send invite)




License
-------

The project is licensed under the MIT license.
