---
title: "Getting started with React Native & Django authentication - Part 2"
date: 2018-05-15T10:46:44+01:00
draft: false
tags: [
    "react-native",
    "django",
    "auth",
    "development"
]
---

In [part 1](https://afdezl.github.io/post/authentication-react-native-django-1) we left off with a working authentication API in Django, we will resume here with the creation of our React Native application.


## 2. Initialising the React Native app
### 2.1 Required toolset

You will need to have node and npm installed. Navigate to the `django-rn-auth` directory set up in part 1 and initlialise a new React Native app as follows:

```bash
npm install -g create-react-native-app
create-react-native-app authapp && cd authapp
```

Once the application directory has been initialised, you can start the development server by running:

```bash
npm start
```

This will display a QR code on the terminal, you will need to download the [Expo](https://play.google.com/store/apps/details?id=host.exp.exponent&hl=en_GB) application on your phone and scan the QR code to see you application (both your phone and laptop need to be connected to same network).

### 2.2 Application layout

We will begin by initialising a `authapp/src` directory and subdirectories that will contain all our application code and components.

```bash
mkdir -p ./src/components
```

Our application will consist of three screens:

* 1 - Login screen
* 2 - Registration screen
* 3 - Home screen (will serve as the logout screen)

For this we will be using the _**react-native-router-flux**_ package, based on react navigation.

```bash
npm install --save react-native-router-flux
```


In order to issue the network requests, React Native provides access to the [*Fetch API*](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) out of the box, instead we will be using the [axios](https://github.com/axios/axios) package as it has some neater features out of the box, such as automatic transformation of JSON data.

```bash
npm install --save axios
```

Ok, we're ready to start writing some React Native at this point.

#### 2.2.1 Routing

We will begin by defining how the user will navigate through the application. Create a **Router.js** file inside the **.src/** directory:

```js
// src/Router.js 

import React from 'react';
import { Scene, Stack, Router, Actions } from 'react-native-router-flux';
import Login from './components/Login';
import Register from './components/Register';
import Home from './components/Home';


const RouterComponent = () => {
  return (
    <Router>
      <Stack hideNavBar key="root">
        <Stack
          key="auth"
          type="reset"
        >
          <Scene
            title="Sign In"
            key="login"
            component={Login}
            initial
          />
          <Scene
            title="Register"
            key="register"
            component={Register}
          />  
        </Stack>
        <Stack
          key="main"
          type="reset"
        >
          <Scene
            title="Home"
            key="home"
            component={Home}
            initial
          />
        </Stack>
      </Stack>
    </Router>
  );
};


export default RouterComponent;
```

If you have issues properly displaying the title and/or location of the navigation bar, you will need to do some styling on the components.

Add the required imports:

```js
import { StyleSheet, StatusBar } from 'react-native';
```

Add the style properties:
```js
const style = StyleSheet.create({
  navBarStyle: {
    top: StatusBar.currentHeight
  },
  titleStyle: {
    flexDirection: 'row',
    width: 200
  }
});
```

Modify the **auth** and **main** `Stacks`: 
```js
<Stack ...>
    <Stack
      key="auth"
      ...
      navigationBarStyle={style.navBarStyle}
      titleStyle={style.titleStyle}
    >
    <Stack
      key="main"
      ...
      navigationBarStyle={style.navBarStyle}
      titleStyle={style.titleStyle}
    >
</Stack>
```


#### 2.2.2 Screen components

Let's now add the three components that will embody our three main screens. These will for now be _dummy_ components to get something going. Create the following inside the **src/components** directory:

* Login

```js
// src/components/Login.js

import React, { Component } from 'react';
import { View, Text } from 'react-native';


class Login extends Component {
  render() {
    return (
      <View>
        <Text>Login screen</Text>
      </View>
    );
  }
}

export default Login;
```

* Registration

```js
// src/components/Register.js

import React, { Component } from 'react';
import { View, Text } from 'react-native';


class Register extends Component {
  render() {
    return (
      <View>
        <Text>Registration screen</Text>
      </View>
    );
  }
}

export default Register;
```

* Home

```js
// src/components/Home.js

import React, { Component } from 'react';
import { View, Text } from 'react-native';


class Home extends Component {
  render() {
    return (
      <View>
        <Text>Home screen</Text>
      </View>
    );
  }
}

export default Home;
```


One last thing we will need to do in order to get this working is replacing the contents of the default `App.js` in the root of our project. We need to indicate that the router component will be handling what is being displayed: 

```js
// App.js

import React, { Component } from 'react';
import Router from './src/Router';


export default class App extends Component {
  render() {
    return (
      <Router />
    );
  }
}
```

This should be enough the application to a working state with routing.


### 2.3 Defining the main components
#### 2.3.1 Login and register components

_**NOTE**: This is the ideal scenario for a [redux](https://github.com/reduxjs/react-redux) implementation in order to share the state between components. For the sake of simplifying this example we will not be using it._ 

There are a number of similitudes between these two components. Both will get some input data, maybe do some processing with it and issue a `POST` request to our API *register* or *login* endpoints.
 It seems sensible then to create a common component that can be used in both the Login and Register components. We will achieve this by passing a `create` prop into the component that will add a couple of extra fields and change the behaviour and look of the submit button.

 Create a subdirectory `src/components/common/` and touch a file called `LoginOrCreateForm.js`. Our common component will look like this:

 ```js
 // src/components/common/LoginOrCreateForm.js

import React, { Component } from 'react';
import { Button, View, Text, TextInput, StyleSheet } from 'react-native';
import { Actions } from 'react-native-router-flux';
import axios from 'axios';


class LoginOrCreateForm extends Component {
  state = {
    username: '',
    password: '',
    firstName: '',
    lastName: ''
  }

  onUsernameChange(text) {
    this.setState({ username: text });
  }

  onPasswordChange(text) {
    this.setState({ password: text });
  }

  onFirstNameChange(text) {
    this.setState({ firstName: text });
  }

  onLastNameChange(text) {
    this.setState({ lastName: text });
  }

  renderCreateForm() {
    const { fieldStyle, textInputStyle } = style;
    if (this.props.create) {
      return (
          <View style={fieldStyle}>
            <TextInput
              placeholder="First name"
              autoCorrect={false}
              onChangeText={this.onFirstNameChange.bind(this)}
              style={textInputStyle}
            />
            <TextInput
              placeholder="Last name"
              autoCorrect={false}
              onChangeText={this.onLastNameChange.bind(this)}
              style={textInputStyle}
            />
          </View>
      );
    }
  }

  renderButton() {
    const buttonText = this.props.create ? 'Create' : 'Login';

    return (
      <Button title={buttonText} onPress={this.handleRequest.bind(this)}/>
    );
  }


  renderCreateLink() {
    if (!this.props.create) {
      const { accountCreateTextStyle } = style;
      return (
        <Text style={accountCreateTextStyle}>
          Or 
          <Text style={{ color: 'blue' }} onPress={() => Actions.register()}>
            {' Sign-up'}
          </Text>
        </Text>
      );
    }
  }

  render() {
    const {
      formContainerStyle,
      fieldStyle,
      textInputStyle,
      buttonContainerStyle,
      accountCreateContainerStyle
    } = style;

    return (
      <View style={{ flex: 1, backgroundColor: 'white' }}>
        <View style={formContainerStyle}>
          <View style={fieldStyle}>
            <TextInput
              placeholder="username"
              autoCorrect={false}
              autoCapitalize="none"
              onChangeText={this.onUsernameChange.bind(this)}
              style={textInputStyle}
            />
          </View>
          <View style={fieldStyle}>
            <TextInput
              secureTextEntry
              autoCapitalize="none"
              autoCorrect={false}
              placeholder="password"
              onChangeText={this.onPasswordChange.bind(this)}
              style={textInputStyle}
            />
          </View>
          {this.renderCreateForm()}
        </View>
        <View style={buttonContainerStyle}>
          {this.renderButton()}
          <View style={accountCreateContainerStyle}>
            {this.renderCreateLink()}
          </View>
        </View>
      </View>
    );
  }
}


const style = StyleSheet.create({
  formContainerStyle: {
    flex: 1,
    flexDirection: 'column',
    alignItems: 'center',
    justifyContent: 'center',
  },
  textInputStyle: {
    flex: 1,
    padding: 15
  },
  fieldStyle: {
    flexDirection: 'row',
    justifyContent: 'center'
  },
  buttonContainerStyle: {
    flex: 1,
    justifyContent: 'center',
    padding: 25
  },
  accountCreateTextStyle: {
    color: 'black'
  },
  accountCreateContainerStyle: {
    padding: 25,
    alignItems: 'center'
  }
});


export default LoginOrCreateForm;

```

We can now replace the contents of our `Login` and `Register` components as so:

```js
// src/components/Login.js

import React, { Component } from 'react';
import { View, Text } from 'react-native';
import LoginOrCreateForm from './common/LoginOrCreateForm';


class Login extends Component {
  render() {
    return (
      <View style={{ flex: 1 }}>
        <LoginOrCreateForm />
      </View>
    );
  }
}

export default Login;
```

```js
// src/components/Register.js

import React, { Component } from 'react';
import { View, Text } from 'react-native';
import LoginOrCreateForm from './common/LoginOrCreateForm';

class Register extends Component {
  render() {
    return (
      <View style={{ flex: 1 }}>
        <LoginOrCreateForm create/>
      </View>
    );
  }
}

export default Register;
```



As you can see, when the button is pressed nothing happens. Here is where we get to the meat of this example. We will add an `onPress` handler that will post the requested data to our backend endpoint and redirect to the Home screen if the response is successfull.

Add the following method to the `LoginOrCreateForm` class:

```js
handleRequest() {
  const endpoint = this.props.create ? 'register' : 'login';
  const payload = { username: this.state.username, password: this.state.password } 
  
  if (this.props.create) {
    payload.first_name = this.state.firstName;
    payload.last_name = this.state.lastName;
  }
  
  axios
    .post(`/auth/${endpoint}/`, payload)
    .then(response => {
      const { token, user } = response.data;

      // We set the returned token as the default authorization header
      axios.defaults.headers.common.Authorization = `Token ${token}`;
      
      // Navigate to the home screen
      Actions.main();
    })
    .catch(error => console.log(error));
}
```

And hook it up with the button's `onPress` handler:

```js
renderButton() {
    const buttonText = this.props.create ? 'Create' : 'Login';

    return (
      <Button title={buttonText} onPress={this.handleRequest.bind(this)}/>
    );
  }
```

Although it can be done, note that we are not passing the entire URL in the request. Axios has a few tricks to help us manage its configuration.

We can set them as defaults upon application load. Navigate to the **App.js** file in the root directory and add the following method to the `App` class:

```js
componentWillMount() {
    axios.defaults.baseURL = 'http://<YOUR_LOCAL_IP>:8000/api';
    axios.defaults.timeout = 1500;
  }
```

Remember to replace `<YOUR_LOCAL_IP>` with your actual local IP address, and if for some reason you're running the django server on a different port, change this as well.

At this point you should be able to start your backend Django server with your local IP and login via the app. Navigate to the **backend** directory set up in [part 1](https://afdezl.github.io/post/authentication-react-native-django-1) and start the server as follows (_remember to activate your virtualenv_):

```bash
python manage.py runserver <YOUR_LOCAL_IP>:8000
```

And replace `<YOUR_LOCAL_IP>` for your local IP.

You should be able to login or register and access the home screen.

#### 2.3.2 Logout component

The logout component will simply consist of a button with a callback to our logout endpoint, upon successful response, the user will be redirected to the login screen:

```js
import React, { Component } from 'react';
import { View, Button, StyleSheet } from 'react-native';
import { Actions } from 'react-native-router-flux';
import axios from 'axios';


class Home extends Component {

  handleRequest() {
    // This request will only succeed if the Authorization header
    // contains the API token
    axios
      .get('/auth/logout/')
      .then(response => {
        Actions.auth()
      })
      .catch(error =>  console.log(error));
  }

  render() {
    const { buttonContainerStyle } = styles;
    return (
      <View style={buttonContainerStyle}>
        <Button title="Logout" onPress={this.handleRequest.bind(this)}/>
      </View>
    );
  }
}


const styles = StyleSheet.create({
  buttonContainerStyle: {
    flex: 1,
    flexDirection: 'column',
    justifyContent: 'center',
    backgroundColor: 'white'
  }
});

export default Home;
```

And voilà!

_NOTE: This is a very simplified version of what authentication with React Native and Django looks like. We have opted for a simple yet comprehensive approach and have compromised on multiple aspects that SHOULD be taken into consideration when creating a production ready application, find a short list of these as follows:_

* _Field validation_
* _Application state management_
* _Error handling_
* _Response status code handling_
* _Endpoint security_
* _Throttling_


All the code for this project can be found in this [repo](https://github.com/afdezl/django-rn-auth)
