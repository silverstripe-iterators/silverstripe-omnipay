# SilverStripe Payments via Omnipay

[![Build Status](https://api.travis-ci.org/burnbright/silverstripe-omnipay.png)](https://travis-ci.org/burnbright/silverstripe-omnipay)
[![Code Coverage](https://scrutinizer-ci.com/g/burnbright/silverstripe-omnipay/badges/coverage.png?s=90fe071f1fec0564c6ee8db6678a73ae5aca9207)](https://scrutinizer-ci.com/g/burnbright/silverstripe-omnipay/)
[![Scrutinizer Quality Score](https://scrutinizer-ci.com/g/burnbright/silverstripe-omnipay/badges/quality-score.png?s=35408c4d68a36e35d0bc2b4b012bd8fb2c6d4d49)](https://scrutinizer-ci.com/g/burnbright/silverstripe-omnipay/)
[![Latest Stable Version](https://poser.pugx.org/burnbright/silverstripe-omnipay/v/stable.png)](https://packagist.org/packages/burnbright/silverstripe-omnipay)
[![Total Downloads](https://poser.pugx.org/burnbright/silverstripe-omnipay/downloads.png)](https://packagist.org/packages/burnbright/silverstripe-omnipay)
[![Latest Unstable Version](https://poser.pugx.org/burnbright/silverstripe-omnipay/v/unstable.png)](https://packagist.org/packages/burnbright/silverstripe-omnipay)

The aim of this module is to make it easy for developers to eaisly integrate the ability to pay for things with their SilverStripe application.
There are many gateway options to choose from, and integrating with additional gateways has a structured approach that should be understandable.
A high quality, simple to use payment module will help to boost the SilverStripe ecosystem, as it allows applications to be profitable.

This module is a complete rewrite of the past Payment module. It is not backwards-compatible.
In a nutshell, it provides a thin wrapping of the PHP Omnipay payments library.
To understand more about omnipay, see: https://github.com/adrianmacneil/omnipay

## Requirements

 * [silverstripe framework](https://github.com/silverstripe/silverstripe-framework) 3.1+
 * [omnipay](https://github.com/omnipay/omnipay) + it's dependencies - which include guzzle and some symphony libraries.

## Features

 * Gateway configuration via yaml config.
 * Payment / transaction model handling.
 * Detailed + structured logging in the database.
 * Provide visitors with one, or many gateways to choose from.
 * Provides form fields, which can change per-gateway.
 * Caters for different types of gateways: on-site capturing, off-site capturing, and manual payment.
 * Wraps the Omnipay php library.
 * Multiple currencies.

## Installation

[Composer](http://doc.silverstripe.org/framework/en/installation/composer) is currently the only supported way to set up this module:
```
composer require burnbright/silverstripe-omnipay
```

## Configuration

You can configure gateway settings in your `mysite/_config/payment.yml` file. Here you can select a list of allowed gateways, and separately set the gateway-specific settings.

You can also choose to enable file logging by setting `file_logging` to 1.

```yaml
---
Name: payment
---
Payment:
    allowed_gateways:
        - 'PayPal_Express'
        - 'PaymentExpress_PxPay'
        - 'Manual'
    parameters:
        PayPal_Express:
            username: 'example.username.test'
            password: 'txjjllae802325'
            signature: 'wk32hkimhacsdfa'
        PaymentExpress_PxPost:
            username: 'EXAMPLEUSER'
            password: '235llgwxle4tol23l'
---
Except:
    environment: 'live'
---
Payment:
    file_logging: 1
    allowed_gateways:
        - 'Dummy'
---
Only:
    environment: 'live'
---
Payment:
    parameters:
        PayPal_Express:
            username: 'liveexample.test'
            password: 'livepassawe23'
            signature: 'laivfe23235235'
        PaymentExpress_PxPost:
            username: 'LIVEUSER'
            password: 'n23nl2ltwlwjle'
---
```

The [SilverStripe documentation](http://doc.silverstripe.com/framework/en/topics/configuration#setting-configuration-via-yaml-files) explains more about yaml config files.

## Usage

### List available gateways

In your application, you may want to allow users to choose between a few different payment gateways. This can be useful for users if their first attempt is declined.

```php
$gateways = GatewayInfo::get_supported_gateways();
```

If no allowed gateways are configured, then the module will default to providing
the "Manual" gateway.

### Get payment form fields

The `GatewayFieldsFactory` helper class enables you to produce a list of appropriately configured form fields for the given gateway.

```php
$factory = new GatewayFieldsFactory($gateway);
$fields = $factory->getFields();
```

If the gateway is off-site, then no credit-card fields will be returned.

Fields have also been appropriately grouped, incase you only want to retrieve the credit card related fields, for example.

Required fields can be configured in the yaml config file, as this information is unfortunately not provided by omnipay:

```yaml
---
Name: payment
---
Payment:
    allowed_gateways:
        - 'PaymentExpress_PxPost'
    parameters:
        PaymentExpress_PxPost:
            username: 'EXAMPLEUSER'
            password: '235llgwxle4tol23l'
                required_fields:
                        - 'issueNumber'
                        - 'startMonth'
                        - 'startYear'
```

### Make a purchase

Using function chaining, we can create and configure a new payment object, and submit a request to the chosen gateway. The response object has a `redirect` function built in that will either redirect the user to the external gateway site, or to the given return url.

```php
    $payment = Payment::create()->init("PxPayGateway", 100, "NZD");
    $response = PurchaseService::create($payment)
        ->setSuccessUrl($this->Link('complete')."/".$donation->ID)
        ->setFailureUrl($this->Link()."?message=payment cancelled")
        ->purchase($form->getData());
    $response->redirect();
```

Of course you don't need to chain all of these functions, as you may want to redirect somewhere else, or do some further setup.

After payment has been made, the user will be redirected to the given return url (or cancel url, if they cancelled).

### onCaptured hook

To call your custom code when returning from an off-site gateway, you'll need to
introduce an extension that utilises the onCaptured extension point.

For example:
```php
class ShopPayment extends DataExtension {

    private static $has_one = array(
        'Order' => 'Order'
    );

    public function onCaptured($response){
        $order = $this->owner->Order();
        $order->completePayment($this->owner);
    }

}
```

## Debugging payments

A useful way to debug payment issues is to enable file logging:

```yaml
---
Name: payment
---
Payment:
    file_logging: true #or use 'verbose' for more detailed output
```

## Caveats and Troubleshooting

Logs will be saved to `debug.log` in the root of your SilverStripe directory.

 * Payments have both an amount and a currency. That means you should check if all currencies match if you will be adding them up.

## Further Documentation

https://github.com/burnbright/silverstripe-omnipay/blob/master/docs/en/index.md