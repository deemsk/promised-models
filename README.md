# Promised Models (in progress)

## Key features

 * promise based
 * typed attributes
 * nested models and collections
 * async calculations and validation

## Install

    $npm install --save promised-models

## Usage

```js
var Model = require('promises-models'),
    FashionModel = new Model.inherit({
        attributes: {
            name: Model.attributeTypes.String
        }
    }),
    model = new FashionModel({
        name: 'Kate'
    });

model.get('name'); // 'Kate'
```


## Api reference (in progress)

### Model sync methods

#### inherit `Model.inherit(properties, [classPorperties])`

Creates you own model class by extending `Model`. You can define attributes, instance/class method and properties. Inheritance is build over [inherit](https://www.npmjs.com/package/inherit).

```
var CountedModels = Model.inherit({
    __constructor: function () {
        this.__base.apply(this, arguments); //super
        this.attributes.index.set(this.__self._count); //static properties
        this.__self._count ++;
    },
    getIndex: function () {
        return this.get('index');
    }
}, {
    _count: 0,
    getCount: function () {
        return this._count;
    }
});
```

#### attributeTypes `Model.attributeTypes`

Namespace for predefined types of attributes. Supported types:

* `String`
* `Number`
* `Boolean`
* `List` — for storing arrays
* `Model` — for nested models
* `ModelList` — for nested collections

You can extend default attribute types or create your own

```
var DateAttribute = Model.attributeTypes.Number.inherit({
    //..
}),
FashionModel = Model.inherit({
    attributes: {
        name: Model.attributeTypes.String,
        birthDate: DateAttribute
    }
});
```

**Note:** `models.attributes` will be replaced in constructor with attribute instances.

```
var model = new FashionModel();
model.attributes.birthDate instanceof DateAttribute; //true
```

#### set `model.set(attributeName, value)`

Set current value of attribute.

```
var model = new FashionModel();
model.set('name', 'Kate');
model.attributes.name.set('Kate');
model.set({
    name: 'Kate',
    birthDate: new Date(1974, 1, 16)
});
```

#### get `model.get(attributeName)`

Get current value of attribute.

```
var model = new FashionModel({
    name: 'Kate',
    birthDate: new Date(1974, 1, 16)
})
model.get('name'); //Kate
model.attributes.name.get(); //Kate
model.get('some'); //throws error as uknown attribute
```

#### toJSON `model.toJSON()`

Return shallow copy of model data.

**Note:** You can create internal attributes, wich wouldn't be included to returned object.

```
var FashionModel = new Model.inherit({
    attributes: {
        name: Model.attributeTypes.String.inherit({
            internal: true;
        }),
        sename: Model.attributeTypes.String.inherit({
            internal: true;
        }),
        fullName: Model.attributeTypes.String
    }
}),
model = new FashionModel({
    name: 'Kate',
    sename: 'Moss',
    fullName: 'Kate Moss'
});
model.toJSON(); // {fullName: 'Kate Moss'}
model.get('name'); // Kate
```

#### isChanged `model.isChanged([branch])`

Has model changed since init or last commit/save/fetch.

```
var FashionModel = Model.inherit({
        attributes: {
            name: Model.attributeTypes.String,
            weight: Model.attributeTypes.Number.inherit({
                default: 50
            })
        }
    }),
    model = new FashionModel({
        name: 'Kate',
        weight: 55
    });
model.isChanged(); //false
model.set('weight', 56);
model.isChanged(); //true
```

#### commit `model.commit([branch])`

Cache current model state

```
var model = new FashionModel();
model.set({
    name: 'Kate',
    weight: 55
});
model.isChanged();//true
model.commit();
model.isChanged();//false
```

#### revert `model.revert([branch])`

Revert model state to last cashed one

```
var model = new FashionModel({
    name: 'Kate',
    weight: 55
});
model.set('weight', 56);
model.revert();
model.get('weight'); //55
model.isChanged(); //false
```

**Note:** You can create your own cache by passing branch param.

```
var RENDERED = 'RENDERED';
model.on('change', function () {
    if (model.isChanged(RENDERED)) {
        View.render();
        model.commit(RENDERED);
    }
});
```

#### on `model.on(events, cb, [ctx])`

Add event handler for one or multiple model events.

List of events:

* `change` – some of attributes have been changed
* `change:attributeName` – `attributeName` have been changed
* `destruct` – model was destructed
* `calculate` – async calculations started

```
model.on('change', this.changeHandler, this);
model.on('change:weight change:name', this.changeHandler, this);
```

#### un `model.un(events, cb, [ctx])`

Unsubscribe event handler from events.

```
//sunscribe
model.on('change:weight change:name', this.changeHandler, this);

//unsunscribe
model.un('change:weight change:name', this.changeHandler, this);
```

#### destruct `model.destruct()`

Remove all events handlers from model and removes model from collections

### Model async methods

#### validate `model.validate()`

Validate model attributes.

```
var FashionModel = Model.inherit({
        attributes: {
            name: Model.attributeTypes.String.inherit({
                validate: function () {
                    return $.get('/validateName', {
                        name: this.get()
                    }).then(function () {
                        return true; //valid
                    }, function () {
                        return false; //invalid
                    });
                }
            })
        }
    }),
    model = new FashionModel();

model.validate().fail(function (err) {
    if (err instanceof Model.ValidationError) {
        console.log('Invalid attributes:' + err.attributes.join());
    } else {
        return err;
    }
}).done();
```

#### ready `model.ready()`

Fullfils when all calculations over model finished.

```
var FashionModel = Model.inherit({
        attributes: {
            name: Model.attributeTypes.String,
            ratingIndex: Model.attributeTypes.Number.inherit({
                calculate: function () {
                    return $.get('/rating', {
                        annualFee: this.model.get('annualFee')
                    });
                }
            }),
            annualFee: Model.attributeTypes.Number
        }
    }),
    model = new FashionModel();

model.set('annualFee', 1000000);
model.ready().then(function () {
    model.get('ratingIndex');
}).done();

```

#### fetch `model.fetch()`

Fetch data associlated with model from storage.

```
var FashionModel = Model.inherit({
        attributes: {
            name: Model.attributeTypes.String
        },
        storage: Model.Storage.inherit({
            find: function (model) {
                return $.get('/models', {
                    id: model.id
                });
            }
        })
    }),
    model = new FashionModel(id);

model.fetch().then(function () {
    model.get('name');
}).done();
```

#### save `model.save()`

```
var FashionModel = Model.inherit({
        attributes: {
            name: Model.attributeTypes.String,
            weight: Model.attributeTypes.Number
        },
        storage: Model.Storage.inherit({
            insert: function (model) {
                return $.post('/models', model.toJSON()).then(function (result) {
                    return result.id;
                });
            },
            update: function (model) {
                return $.put('/models', model.toJSON());
            }
        })
    }),
    model = new FashionModel();

model.set({
    name: 'Kate',
    weight: 55
});
model.save().then(function () { //create
    model.id; //storage id
    model.set('weight', 56);
    return model.save(); //update
}).done()
```

#### remove `model.remove()`

Removes model from storage.

### Model additional methods and properties

* `model.isNew()`
* `model.isReady()`
* `model.trigger(event)`
* `model.calculate()`
* `model.CHANGE_BRANCH`
* `model.CALCULATIONS_BRANCH`

This methods provided for advanced model extending. Consult source for detials.

### Model static methods and properties

#### Storage `Model.Storage`

Abstract class for model storage

```
var FashionModel = Model.inherit({
    attributes: {
        //..
    },
    storage: Model.Storage.inherit({
        //..
    })
});
```

#### Attribute `Model.Attribute`

Base class for model attribute

```
var CustomAttribute = Model.attribute.inherit({
    //..
})
```

#### ValidationError `Model.ValidationError`

Error class for validation fail report

## run tests

    $ npm test
