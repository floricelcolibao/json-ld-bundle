# Schema.org JSON-LD Symfony Bundle

Schema.org JSON-LD generator for the Symfony 2.8 and 3.0+.

#### PHP 7 support

As of release 3.3.2 of the https://github.com/secit-pl/schema-org all types and properties should be suffixed.
This bundle allows using both class versions (the new suffixed classes and the deprecated non suffixed), but be aware
that only suffixed classes will work properly on the PHP 7.

Another reason for moving to the new class naming schema is the fact that all non suffixed classes will be removed
in the release 3.4 of the secit-pl/schema-org.

## Installation

From the command line run

```
$ composer require secit-pl/json-ld-bundle
```

Update your AppKernel by adding the bundle declaration

```php
class AppKernel extends Kernel
{
    public function registerBundles()
    {
        $bundles = [
            ...
            new SecIT\JsonLdBundle\JsonLdBundle(),
        ];

        ...
    }
}
```

## Usage

### Basic Usage

First of all you need to create a Transformer which will transform your object to schema.org data mapping.

```php
namespace Test\TestBundle\JsonLd;

use SecIT\JsonLdBundle\Transformer\TransformerInterface;
use SecIT\SchemaOrg\Mapping\DataType\TextType;
use SecIT\SchemaOrg\Mapping\Property\NameProperty;
use SecIT\SchemaOrg\Mapping\Type\ThingType;

class TestTransformer implements TransformerInterface
{
    public function transform($object)
    {
        return (new ThingType())
            ->setName(new NameProperty(
                new TextType($object->getName())
            ));
    }
}
```

As the basis for this of this bundle is the https://github.com/secit-pl/schema-org so the transformer should return the object accepted by the JSON-LD Generator.

Next you need to register the transformer as service using the secit.jsonld_transformer tag in your services.yml.

```yaml
services:
    test.object_transformer:
        class: Test\TestBundle\JsonLd\TestTransformer
        tags:
            - { name: secit.jsonld_transformer, class: Test\TestBundle\Classes\ClassToBeTransformedToJsonLd }
```

If you want to assign more than one class to the same transformer you can add multiple tags to the same service.

```yaml
services:
    test.object_transformer:
        class: Test\TestBundle\JsonLd\TestTransformer
        tags:
            - { name: secit.jsonld_transformer, class: Test\TestBundle\Classes\Class1 }
            - { name: secit.jsonld_transformer, class: Test\TestBundle\Classes\Class2 }
            - { name: secit.jsonld_transformer, class: Test\TestBundle\Classes\Class3 }
            ...
```

From now you can transform the specified in the tag class attribute object (in the following example the \Test\TestBundle\Classes\ClassToBeTransformedToJsonLd) to the JSON-LD as following:
 
```php
$object = new \Test\TestBundle\Classes\ClassToBeTransformedToJsonLd();
$object->setName('Some name');
echo $this->getContainer()->get('secit.json_ld')->generate($object);
```

The output should be something like this:

```html
<script type="application/ld+json">{"@context":"http:\/\/schema.org","@type":"Thing","name":"Some name"}</script>
```

### Advanced usage

In many situations it's required to have a nested transformers to not implement whole logic in the single class.
To use nested transformers your Transformer should implement JsonLdAwareInterface. If you don't want to implement
the interface methods by your own you can use the JsonLdAwareTrait.

Here is the simple example of how to use it:

PersonTransformer.php
```php
namespace Test\TestBundle\JsonLd;

use SecIT\JsonLdBundle\DependencyInjection\JsonLdAwareInterface;
use SecIT\JsonLdBundle\DependencyInjection\JsonLdAwareTrait;
use SecIT\JsonLdBundle\Transformer\TransformerInterface;
use SecIT\SchemaOrg\Mapping\DataType\TextType;
use SecIT\SchemaOrg\Mapping\Property\NameProperty;
use SecIT\SchemaOrg\Mapping\Type\PersonType;

class PersonTransformer implements TransformerInterface
{
    public function transform($person)
    {
        return (new PersonType())
            ->setName(new NameProperty(
                new TextType($person->name)
            ));
    }
}
```

ArticleTransformer.php
```php
namespace Test\TestBundle\JsonLd;

use SecIT\JsonLdBundle\DependencyInjection\JsonLdAwareInterface;
use SecIT\JsonLdBundle\DependencyInjection\JsonLdAwareTrait;
use SecIT\JsonLdBundle\Transformer\TransformerInterface;
use SecIT\SchemaOrg\Mapping\DataType\TextType;
use SecIT\SchemaOrg\Mapping\Property\AuthorProperty;
use SecIT\SchemaOrg\Mapping\Property\NameProperty;
use SecIT\SchemaOrg\Mapping\Type\ArticleType;

class ArticleTransformer implements TransformerInterface, JsonLdAwareInterface
{
    use JsonLdAwareTrait;

    public function transform($article)
    {
        return (new ArticleType())
            ->setName(new NameProperty(
                new TextType($article->name)
            ))
            ->setAuthor(new AuthorProperty(
                $this->getJsonLd()->transform($article->author)
            ));
    }
}
```

Example input object:
```php
$author = new Person();
$author->name = 'Jon Smith';

$article = new Article();
$article->name = 'Example article';
$article->author = $author;
```

The output:
```html
<script type="application/ld+json">{"@context":"http:\/\/schema.org","@type":"Article","name":"Example article","author":{"@type":"Person","name":"Jon Smith"}}</script>
```

### Twig Support

This bundle also provides the Twig extension which allows to render JSON-LD directly from the Twig templates.

TestController:

```php
namespace Test\TestBundle\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;

class TestController extends Controller
{
    public function testAction()
    {
        return $this->render('TestBundle:Test:example.html.twig', [
            'object' => new Foo(),
        ]);
    }
}
```

example.html.twig:
```twig
{{ object|json_ld }}
```

The output:
```html
<script type="application/ld+json">{"@context":"http:\/\/schema.org", ... }</script>
```