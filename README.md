# RESUMEN EXAMEN DE LABORATORIO IISSI 2

## 1. Inicialización
1. Importar repositorios a VSCode con `git clone <URL>`.
2. Instalar dependencias de back y front `npm install`.
3. Realizamos las migraciones a la DB:
    1. Abrimos HeidiSQL y creamos DB nueva con el usuario "iissi_user" y contraseña "iissi$user". Comprobamos puertos de aplicación con el `.env`.
    2. Ejecutamos migraciones y seeders. Ctrl+Shift+P o F1 y `Tasks: run task` y seleccionamos `Rebuild database`.
4. Ejecutamos en backend `npm start`.
5. LEEMOS ENUNCIADO!!

## 2. Backend
### Procedimiento
1. Cambiar `models` y `migrations` conforme enunciado.
2. Añadir nuevo apartado en `Validation`.
    1. HAY QUE ACTUALIZAR TANTO EN CREATE COMO EN UPDATE!!
3. En su caso, añadir nuevo apartado en `Controllers`.

### Laboratorios de backend
1. LAB 2 - Routing & Controllers: https://github.com/IISSI2-IS-2022-2023/Lab2-Backend-Routing-Controllers
2. LAB 3 - Validation & Middleware: https://github.com/IISSI2-IS-2022-2023/Lab3-Backend-Validation-Middleware

#### Ejemplo de Route
```JavaScript
app.route('/products')
    .post(
      middlewares.isLoggedIn,
      middlewares.hasRole('owner'),
      upload,
      middlewares.checkProductRestaurantOwnership,
      ProductValidation.create(),
      ProductController.create
    )
```

### Examen Junio - Tarde:

#### Todos los cambios dentro de MODELS en archivo `product.js`:

```JavaScript
Product.init({
    name: DataTypes.STRING,
    description: DataTypes.STRING,
    fats: DataTypes.DOUBLE, //CAMBIO
    proteins: DataTypes.DOUBLE, //CAMBIO
    carbohydrates: DataTypes.DOUBLE, //CAMBIO
    calories: DataTypes.DOUBLE, //CAMBIO
    price: DataTypes.DOUBLE,
    image: DataTypes.STRING,
    order: DataTypes.INTEGER,
    availability: DataTypes.BOOLEAN,
    restaurantId: DataTypes.INTEGER,
    productCategoryId: DataTypes.INTEGER
  }, {
    sequelize,
    modelName: 'Product'
  })
  return Product
}
```

#### Todos los cambios dentro de MIGRATIONS en archivo `create-product.js`:

```JavaScript
module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('Products', {
      ...
      fats: {
        type: Sequelize.DOUBLE,
        allowNull: false
      },
      proteins: {
        type: Sequelize.DOUBLE,
        allowNull: false
      },
      carbohydrates: {
        type: Sequelize.DOUBLE,
        allowNull: false
      },
      calories: {
        type: Sequelize.DOUBLE,
        allowNull: false
      },
      ...
    })
  },
  down: async (queryInterface, Sequelize) => {
    await queryInterface.dropTable('Products')
  }
}
```

#### Todos los cambios dentro de la VALIDATIONS en archivo `ProductValidation.js`:

```JavaScript
const calculateCalories = (fats, proteins, carbohydrates) => {
  fats = parseFloat(fats)
  proteins = parseFloat(proteins)
  carbohydrates = parseFloat(carbohydrates)

  const calories = 9 * fats + 4 * proteins + 4 * carbohydrates

  if (calories > 1000 || fats + proteins + carbohydrates !== 100) {
    return false
  } else {
    return true
  }
}

module.exports = {
    create: () => {
        return [
            ...
            check('fats')
            .custom((value, { req }) => {
                return calculateCalories(value, req.body.proteins, req.body.carbohydrates)
            })
            .withMessage('Un alimento no puede superar las 1000 calorias por cada 100 gramos.')
        ]
    },
    update: () => {
        return [
            ...
            check('fats')
            .custom((value, { req }) => {
                return calculateCalories(value, req.body.proteins, req.body.carbohydrates)
            })
            .withMessage('Un alimento no puede superar las 1000 calorias por cada 100 gramos.')
        ]
    }
}
        
```

#### En CONTROLLERS no se realiza ningún cambio.

### Examen Junio - Mañana:

#### Todos los cambios dentro de MODELS en archivo `restaurant.js`:

```JavaScript
Restaurant.init({
    name: DataTypes.STRING,
    promoted: DataTypes.BOOLEAN, //CAMBIO
    description: DataTypes.TEXT,
    ...
  }, {
    sequelize,
    modelName: 'Restaurant'
  })
  return Restaurant
}

```

#### Todos los cambios dentro de MIGRATIONS en archivo `create-restaurant.js`:

```JavaScript
module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('Restaurants', {
      ...
      promoted: { //CAMBIO
        allowNull: false,
        type: Sequelize.BOOLEAN,
        defaultValue: true
      },
      ...
    })
  },
  down: async (queryInterface, Sequelize) => {
    await queryInterface.dropTable('Restaurants')
  }
}

```

#### Todos los cambios dentro de VALIDATIONS en archivo `RestaurantValidation.js`:

```JavaScript
const promotedRestaurant = async (ownerId, isPromoted) => {
  let ruleBroken = false
  if (isPromoted === true) {
    try {
      const ownerRestaurants = await Restaurant.findAll({
        where: {
          userId: ownerId,
          promoted: true
        }
      })
      if (isPromoted && ownerRestaurants.lenght > 0) {
        ruleBroken = true
      }
    } catch (error) {
      ruleBroken = true
    }
  }
  return ruleBroken ? Promise.reject(new Error('You can only promote one restaurant at a time')) : Promise.resolve()
}

module.exports = {
    create: () => {
        return [
            ...
            check('promoted')
            .custom((value, { req }) => {
                return promotedRestaurant(req.user.id, value)
            })
            .withMessage('You cant promote more than one restaurant at the same time.')
        ]
    },
    update: () => {
        return [
            ...
            check('promoted')
            .custom((value, { req }) => {
                return promotedRestaurant(req.user.id, value)
            })
            .withMessage('You cant promote more than one restaurant at the same time.')
        ]
    }
}
```

#### Todos los cambios dentro de CONTROLLERS en archivo `RestaurantController.js`:

```JavaScript
exports.show = async function (req, res) {
  // Only returns PUBLIC information of restaurants
  try {
    const restaurant = await Restaurant.findByPk(req.params.restaurantId, {
      attributes: { exclude: ['userId'] },
      include: [{
        model: Product,
        as: 'products',
        order: [['promoted', 'DESC']], //ESTOS SON LOS CAMBIOS
        include: { model: ProductCategory, as: 'productCategory' }
      },
      {
        model: RestaurantCategory,
        as: 'restaurantCategory'
      }]
    }
    )
    res.json(restaurant)
  } catch (err) {
    res.status(404).send(err)
  }
}
```

### 123.Cambios en validacion:
Links a lab:
Lo hago porque se trata de un textInput
```JavaScript
const checkSum100 = (fats, proteins, carbohydrates) => {
  fats = parseFloat(fats)
  proteins = parseFloat(proteins)
  carbohydrates = parseFloat(carbohydrates)

  if ((fats < 0 || proteins < 0 || carbohydrates < 0) || (fats + proteins + carbohydrates) !== 100) {
    return false
  }

  return true
}

// Solution
const checkCalories = (fats, proteins, carbohydrates) => {
  fats = parseFloat(fats)
  proteins = parseFloat(proteins)
  carbohydrates = parseFloat(carbohydrates)

  const calories = fats * 9 + proteins * 4 + carbohydrates * 4

  return calories <= 1000
}

 check('fats').custom((values, { req }) => {
        const { fats, proteins, carbohydrates } = req.body
        return checkSum100(fats, proteins, carbohydrates)
      }).withMessage('The values of fat, protein and carbohydrates must be in the range [0, 100] and the sum must be 100.'),
      check('fats').custom((values, { req }) => {
        const { fats, proteins, carbohydrates } = req.body
        return checkCalories(fats, proteins, carbohydrates)
      }).withMessage('The number of calories must not be greater than 1000.')

```

#### Solo un promoted sino no puedo crear
```JavaScript
const checkBusinessRuleOneResturantPromotedByOwner = async (ownerId, promotedValue) => {
  let isBusinessRuleToBeBroken = false
  if (promotedValue === 'true') {
    try {
      const promotedRestaurants = await Restaurant.findAll({ where: { userId: ownerId, promoted: true } })
      if (promotedRestaurants.length !== 0) {
        isBusinessRuleToBeBroken = true
      }
    } catch (error) {
      isBusinessRuleToBeBroken = true
    }
  }
  return isBusinessRuleToBeBroken ? Promise.reject(new Error('Only one promoted per owner')) : Promise.resolve()
} 

 check('promoted')
        .custom(async (value, { req }) => {
          return checkBusinessRuleOneResturantPromotedByOwner(req.user.id, value)
        })
        .withMessage('You can only promote one restaurant at a time')
```

### Cambios raros en controllers

#### Transaccion no puede haber dos al mismo tiempo, si hay cambiar
```JavaScript
exports.promote = async function (req, res) {
  const t = await models.sequelize.transaction()
  try {
    const existingPromotedRestaurant = await Restaurant.findOne({ where: { userId: req.user.id, promoted: true } })
    if (existingPromotedRestaurant) {
      await Restaurant.update(
        { promoted: false },
        { where: { id: existingPromotedRestaurant.id } },
        { transaction: t }
      )
    }
    await Restaurant.update(
      { promoted: true },
      { where: { id: req.params.restaurantId } },
      { transaction: t }
    )
    await t.commit()
    const updatedRestaurant = await Restaurant.findByPk(req.params.restaurantId)
    res.json(updatedRestaurant)
  } catch (err) {
    await t.rollback()
    res.status(500).send(err)
  }
  ```
 #### Comprobar precio medio
 ```JavaScript
 const updateRestaurantInexpensiveness = async function (restaurantId) {
  const resultOtherRestaurants = await Product.findAll({
    where: {
      restaurantId: { [Sequelize.Op.ne]: restaurantId }
    },
    attributes: [
      [Sequelize.fn('AVG', Sequelize.col('price')), 'computedAvgPrice']
    ]
  })
  const resultCurrentRestaurant = await Product.findAll({
    where: {
      restaurantId: restaurantId
    },
    attributes: [
      [Sequelize.fn('AVG', Sequelize.col('price')), 'computedAvgPrice']
    ]
  })
  const avgPriceOtherRestaurants = resultOtherRestaurants[0].dataValues.computedAvgPrice
  const avgPriceCurrentRestaurant = resultCurrentRestaurant[0].dataValues.computedAvgPrice
  const isInexpensive = avgPriceCurrentRestaurant < avgPriceOtherRestaurants
  Restaurant.update({ isInexpensive: isInexpensive }, { where: { id: restaurantId } })
} 
```


## 3. Frontend

### Procedimiento

### Laboratorios de Frontend
1. LAB 4 - Setup & Navigation: https://github.com/IISSI2-IS-2022-2023/Lab4-FrontEnd-ReactNativeBasics
2. LAB 5 - ReactNative Basics: https://github.com/IISSI2-IS-2022-2023/Lab5-FrontEnd-RestfulAPI-Queries
3. LAB 6 - RestfulAPI & Queries: https://github.com/IISSI2-IS-2022-2023/Lab6-FrontEnd-FlexLayout-Forms
4. LAB 7 - FlexLayout & Forms: https://github.com/IISSI2-IS-2022-2023/Lab7-FrontEnd-FormsValidation-POSTRequests
5. LAB 8 - Forms Validation & POST Requests: https://github.com/IISSI2-IS-2022-2023/Lab8-FrontEnd-PUT-DELETE-Forms-Requests

#### Ejemplo de EndPoints `RestaurantEndPoints.js`
```JavaScript
import { get, post } from './helpers/ApiRequestsHelper'
function getAll () {
  return get('users/myrestaurants')
}

function getDetail (id) {
  return get(`restaurants/${id}`)
}

function getRestaurantCategories () {
  return get('restaurantCategories')
}

function create (data) {
  return post('restaurants', data)
}

export { getAll, getDetail, getRestaurantCategories, create }
```
### Examen prueba
```JavaScript
  const [restaurantToBePromoted, setRestaurantToBePromoted] = useState(null)
  
<View style={{ flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center' }} >
              <TextSemiBold>Shipping: <TextSemiBold textStyle={{ color: GlobalStyles.brandPrimary }}>{item.shippingCosts.toFixed(2)}€</TextSemiBold></TextSemiBold>
              {item.promoted &&
                  <TextRegular textStyle={[styles.badge, { color: GlobalStyles.brandPrimary, borderColor: GlobalStyles.brandSuccess }] }>¡En promoción!</TextRegular>
              }
          </View>
 
 <Pressable
              onPress={() => { setRestaurantToBePromoted(item) }}
              style={({ pressed }) => [
                {
                  backgroundColor: pressed
                    ? GlobalStyles.brandSuccessTap
                    : GlobalStyles.brandSuccess
                },
                styles.actionButton
              ]}>
            <View style={[{ flex: 1, flexDirection: 'row', justifyContent: 'center' }]}>
              <MaterialCommunityIcons name='alert-octagram' color={'white'} size={20}/>
              <TextRegular textStyle={styles.text}>
                Promote
              </TextRegular>
            </View>
          </Pressable>
          
     const promoteRestaurant = async (restaurant) => {
    try {
      await promote(restaurant.id)
      await fetchRestaurants()
      setRestaurantToBePromoted(null)
      showMessage({
        message: `Restaurant ${restaurant.name} succesfully promoted`,
        type: 'success',
        style: GlobalStyles.flashStyle,
        titleStyle: GlobalStyles.flashTextStyle
      })
    } catch (error) {
      console.log(error)
      setRestaurantToBePromoted(null)
      showMessage({
        message: `Restaurant ${restaurant.name} could not be promoted.`,
        type: 'error',
        style: GlobalStyles.flashStyle,
        titleStyle: GlobalStyles.flashTextStyle
      })
    }
  }
  return(
   <ConfirmationModal
      isVisible={restaurantToBePromoted !== null}
      onCancel={() => setRestaurantToBePromoted(null)}
      onConfirm={() => promoteRestaurant(restaurantToBePromoted)}>
        <TextRegular>Other promoted restaurant, if any, will be unpromoted</TextRegular>
    </ConfirmationModal>)
    
     styles.badge: {
    textAlign: 'center',
    borderWidth: 2,
    paddingHorizontal: 10,
    borderRadius: 10
  }
          
 ``` 
### Examen Junio - Tarde:

#### Todos los cambios dentro de SCREENS en archivo `CreateProductScreen.js`:

```JavaScript

export default function CreateProductScreen ({ navigation, route }) {
  const [open, setOpen] = useState(false)
  const [productCategories, setProductCategories] = useState([])
  const [backendErrors, setBackendErrors] = useState()

  const initialProductValues = { name: '', description: '', price: 0, order: 0, fats: 0, proteins: 0, carbohydrates: 0, restaurantId: route.params.id, productCategoryId: null, availability: true }
  const validationSchema = yup.object().shape({
    name: yup
      .string()
      .max(30, 'Name too long')
      .required('Name is required'),
    price: yup
      .number()
      .positive('Please provide a positive price value')
      .required('Price is required'),
    order: yup
      .number()
      .positive('Please provide a positive cost value')
      .integer('Please provide an integer cost value'),
    // Solution
    fats: yup
      .number()
      .positive('Please provide a positive cost value')
      .max(100, 'The values must be lower or equal than 100'),
    proteins: yup
      .number()
      .positive('Please provide a positive cost value')
      .max(100, 'The values must be lower or equal than 100'),
    carbohydrates: yup
      .number()
      .positive('Please provide a positive cost value')
      .max(100, 'The values must be lower or equal than 100')
  })

  useEffect(() => {
    ...
    fetchProductCategories()
  }, [])

  const pickImage = async (onSuccess) => {
    ...
  }

  const createProduct = async (values) => {
    setBackendErrors([])
    try {
      console.log(values)
      const createdProduct = await create(values)
      showMessage({
        message: `Product ${createdProduct.name} succesfully created`,
        type: 'success',
        style: flashStyle,
        titleStyle: flashTextStyle
      })
      navigation.navigate('RestaurantDetailScreen', { id: route.params.id, dirty: true })
    } catch (error) {
      console.log(error)
      setBackendErrors(error.errors)
    }
  }
  return (
    <Formik
      validationSchema={validationSchema}
      initialValues={initialProductValues}
      onSubmit={createProduct}>
      {({ handleSubmit, setFieldValue, values }) => (
        <ScrollView>
          <View style={{ alignItems: 'center' }}>
            <View style={{ width: '60%' }}>
              <InputItem
                name='name'
                label='Name:'
              />
              <InputItem
                name='description'
                label='Description:'
              />
              <InputItem
                name='price'
                label='Price:'
              />
              <InputItem
                name='order'
                label='Order/position to be rendered:'
              />
              {/* Solution */}
              <InputItem
                name='fats'
                label='Fats:'
              />
              <InputItem
                name='proteins'
                label='Proteins:'
              />
              <InputItem
                name='carbohydrates'
                label='Carbohydrates:'
              />
              <DropDownPicker
                open={open}
                value={values.productCategoryId}
                items={productCategories}
                setOpen={setOpen}
                onSelectItem={item => {
                  setFieldValue('productCategoryId', item.value)
                }}
                setItems={setProductCategories}
                placeholder="Select the product category"
                containerStyle={{ height: 40, marginTop: 20, marginBottom: 20 }}
                style={{ backgroundColor: brandBackground }}
                dropDownStyle={{ backgroundColor: '#fafafa' }}
              />

              <TextRegular>Is it available?</TextRegular>
              <Switch
                trackColor={{ false: brandSecondary, true: brandPrimary }}
                thumbColor={values.availability ? brandSecondary : '#f4f3f4'}
                // onValueChange={toggleSwitch}
                value={values.availability}
                style={styles.switch}
                onValueChange={value =>
                  setFieldValue('availability', value)
                }
              />

              <Pressable onPress={() =>
                pickImage(
                  async result => {
                    await setFieldValue('image', result)
                  }
                )
              }
                style={styles.imagePicker}
              >
                <TextRegular>Product image: </TextRegular>
                <Image style={styles.image} source={values.image ? { uri: values.image.uri } : defaultProduct} />
              </Pressable>

              {backendErrors &&
                backendErrors.map((error, index) => <TextError key={index}>{error.msg}</TextError>)
              }

              <Pressable
                onPress={ handleSubmit }
                style={({ pressed }) => [
                  {
                    backgroundColor: pressed
                      ? brandPrimaryTap
                      : brandPrimary
                  },
                  styles.button
                ]}>
                <TextRegular textStyle={styles.text}>
                  Create product
                </TextRegular>
              </Pressable>
            </View>
          </View>
        </ScrollView>
      )}
    </Formik>
  )
}

const styles = StyleSheet.create({
  ...
})
```

#### Todos los cambios dentro de SCREENS en archivo `RestaurantDetailScreen.js`:

```JavaScript
export default function RestaurantDetailScreen ({ navigation, route }) {
  const [restaurant, setRestaurant] = useState({})

  useEffect(() => {
    ...
  }, [route])

  const renderHeader = () => {
    return (
      ...
    )
  }

  const renderProduct = ({ item }) => {
    return (
      <ImageCard
        imageUri={item.image ? { uri: process.env.API_BASE_URL + '/' + item.image } : undefined}
        title={item.name}
      >
        <TextRegular numberOfLines={2}>{item.description}</TextRegular>
        <TextSemiBold textStyle={styles.price}>{item.price.toFixed(2)}€</TextSemiBold>
        {/* Solution */}
        {item.fats && <TextSemiBold>Nutritional composition:</TextSemiBold>}
        {item.fats && <View style={{ flexDirection: 'row', paddingLeft: 10 }}><TextSemiBold>Fats: </TextSemiBold> <TextRegular>{item.fats.toFixed(2)}</TextRegular></View>}
        {item.proteins && <View style={{ flexDirection: 'row', paddingLeft: 10 }}><TextSemiBold>Proteins: </TextSemiBold> <TextRegular>{item.proteins.toFixed(2)}</TextRegular></View>}
        {item.carbohydrates && <View style={{ flexDirection: 'row', paddingLeft: 10 }}><TextSemiBold>Carbohydrates: </TextSemiBold> <TextRegular>{item.carbohydrates.toFixed(2)}</TextRegular></View>}
        {item.calories && <View style={{ flexDirection: 'row', paddingLeft: 10 }}><TextSemiBold>Calories: </TextSemiBold> <TextRegular>{item.calories.toFixed(2)}</TextRegular></View>}

      </ImageCard>
    )
  }

  const renderEmptyProductsList = () => {
    ...
  }

  return (
    <View style={styles.container}>
      <FlatList
        ListHeaderComponent={renderHeader}
        ListEmptyComponent={renderEmptyProductsList}
        style={styles.container}
        data={restaurant.products}
        renderItem={renderProduct}
        keyExtractor={item => item.id.toString()}
      />
    </View>
  )
}

const styles = StyleSheet.create({
  ...
})
```

#### Examen Junio - Mañana: 

#### Todos los cambios dentro de SCREENS en archivo `CreateRestaurantScreen.js`:
```JavaScript
export default function CreateRestaurantScreen ({ navigation }) {
  const [open, setOpen] = useState(false)
  const [restaurantCategories, setRestaurantCategories] = useState([])
  const [backendErrors, setBackendErrors] = useState()

  // SOLUTION
  const initialRestaurantValues = { name: '', description: '', address: '', postalCode: '', url: '', shippingCosts: 0, email: '', phone: '', restaurantCategoryId: '', promoted: true }

  const validationSchema = yup.object().shape({
    ...
  })

  useEffect(() => {
    ...
  }, [])

  React.useEffect(() => {
    ...
  }, [])

  const pickImage = async (onSuccess) => {
    ...
  }

  const createRestaurant = async (values) => {
    ...
  }

  return (
    <Formik
      validationSchema={validationSchema}
      initialValues={initialRestaurantValues}
      onSubmit={createRestaurant}>
      {({ handleSubmit, setFieldValue, values }) => (
        <ScrollView>
          <View style={{ alignItems: 'center' }}>
            <View style={{ width: '60%' }}>
              <InputItem
                name='name'
                label='Name:'
              />
              <InputItem
                name='description'
                label='Description:'
              />
              <InputItem
                name='address'
                label='Address:'
              />
              <InputItem
                name='postalCode'
                label='Postal code:'
              />
              <InputItem
                name='url'
                label='Url:'
              />
              <InputItem
                name='shippingCosts'
                label='Shipping costs:'
              />
              <InputItem
                name='email'
                label='Email:'
              />
              <InputItem
                name='phone'
                label='Phone:'
              />

              <DropDownPicker
                open={open}
                value={values.restaurantCategoryId}
                items={restaurantCategories}
                setOpen={setOpen}
                onSelectItem={ item => {
                  setFieldValue('restaurantCategoryId', item.value)
                }}
                setItems={setRestaurantCategories}
                placeholder="Select the restaurant category"
                containerStyle={{ height: 40, marginTop: 20 }}
                style={{ backgroundColor: brandBackground }}
                dropDownStyle={{ backgroundColor: '#fafafa' }}
              />
              <ErrorMessage name={'restaurantCategoryId'} render={msg => <TextError>{msg}</TextError> }/>

              <Pressable onPress={() =>
                pickImage(
                  async result => {
                    await setFieldValue('logo', result)
                  }
                )
              }
                style={styles.imagePicker}
              >
                <TextRegular>Logo: </TextRegular>
                <Image style={styles.image} source={values.logo ? { uri: values.logo.uri } : restaurantLogo} />
              </Pressable>

              <Pressable onPress={() =>
                pickImage(
                  async result => {
                    await setFieldValue('heroImage', result)
                  }
                )
              }
                style={styles.imagePicker}
              >
                <TextRegular>Hero image: </TextRegular>
                <Image style={styles.image} source={values.heroImage ? { uri: values.heroImage.uri } : restaurantBackground} />
              </Pressable>

              {/* SOLUTION */}
              <TextRegular>Is it promoted?</TextRegular>
              <Switch
                trackColor={{ false: brandSecondary, true: brandPrimary }}
                thumbColor={values.promoted ? brandSecondary : '#f4f3f4'}
                value={values.promoted}
                style={styles.switch}
                onValueChange={value =>
                  setFieldValue('promoted', value)
                }
              />

              {backendErrors &&
                backendErrors.map((error, index) => <TextError key={index}>{error.msg}</TextError>)
              }

              <Pressable
                onPress={handleSubmit}
                style={({ pressed }) => [
                  {
                    backgroundColor: pressed
                      ? brandPrimaryTap
                      : brandPrimary
                  },
                  styles.button
                ]}>
                <TextRegular textStyle={styles.text}>
                  Create restaurant
                </TextRegular>
              </Pressable>
            </View>
          </View>
        </ScrollView>
      )}
    </Formik>
  )
}

const styles = StyleSheet.create({
  ...
  // SOLUTION
  switch: {
    marginTop: 5
  }
})
```

#### Todos los cambios dentro de SCREENS en archivo `RestaurantsScreen.js`:

```JavaScript
export default function RestaurantsScreen ({ navigation, route }) {
  const [restaurants, setRestaurants] = useState([])
  const { loggedInUser } = useContext(AuthorizationContext)

  useEffect(() => {
    ...
  }, [loggedInUser, route])

  const renderRestaurant = ({ item }) => {
    return (
      <ImageCard
        imageUri={item.logo ? { uri: process.env.API_BASE_URL + '/' + item.logo } : undefined}
        title={item.name}
        onPress={() => {
          navigation.navigate('RestaurantDetailScreen', { id: item.id })
        }}
      >
         {/* SOLUTION */}
         {item.promoted &&
          <TextRegular textStyle={{ color: brandPrimary, textAlign: 'right' }}>En promoción!</TextRegular>
        }
        <TextRegular numberOfLines={2}>{item.description}</TextRegular>
        {item.averageServiceMinutes !== null &&
          <TextSemiBold>Avg. service time: <TextSemiBold textStyle={{ color: brandPrimary }}>{item.averageServiceMinutes} min.</TextSemiBold></TextSemiBold>
        }
        <TextSemiBold>Shipping: <TextSemiBold textStyle={{ color: brandPrimary }}>{item.shippingCosts.toFixed(2)}€</TextSemiBold></TextSemiBold>
      </ImageCard>
    )
  }

  const renderEmptyRestaurantsList = () => {
    ...
  }

  const renderHeader = () => {
    ...
  }

  return (
    <FlatList
      style={styles.container}
      data={restaurants}
      renderItem={renderRestaurant}
      keyExtractor={item => item.id.toString()}
      ListHeaderComponent={renderHeader}
      ListEmptyComponent={renderEmptyRestaurantsList}
    />
  )
}

const styles = StyleSheet.create({
  ...
})
```


  
 
