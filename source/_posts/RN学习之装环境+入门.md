---
title: RN学习之装环境+入门
date: 2024-07-21 16:48:42
tags: RN
---

## 环境安装

原生环境安装是一个比较麻烦的过程，可以直接使用官网提供的expo环境。

直接使用 HoeBrew，NVM安装，当人如果已经安装了node，可直接使用不用重复安装。

``` sh
nvm install node@18
// 然后npm或者yarn 安装expo-cli
npm global add expo-cli
```

Xcode 的安装直接在APP Store中搜索安装。



新建项目启动：

``` sh
expo init AwesomeProject
cd AwesomeProject
yarn start # you can also use: expo start
```

启动之后根据提示按 ‘i’,会自动启动xcode 中的simulator。

之后进项目工程的APP.js 进行代码编写，语法和React 类似，只不过在编写的过程中要注意一些RN和React的区别，RN类似重写了React的React-dom包，也就是宿主环境相关的一些包被重写了，也就没有了div img 之类的H5 标签，转而代替的是 "View" , "Image"等，并且不可以直接写字符串到标签中，必须使用Text包一层,类似下面这种写法：

```javascript
import { View, Text } from 'react-native'
import React from 'react'

export default function Self() {
  return (
    <View>
      <Text>S</Text>
    </View>
  )
}
```

其中RN的路由导航和React的开发类似，一般借助第三方库来完成"'@react-navigation/native"，具体用法如下所示：

``` javascript
// In App.js in a new project

import * as React from 'react';
import { View, Text } from 'react-native';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import Bootom from '../pages/bottom'
import Detail from '../pages/detail'
const Stack = createNativeStackNavigator();

function Index() {
    return (
        <NavigationContainer>
            <Stack.Navigator screenOptions={{
                headerTitleAlign: 'Center',
                headerTitleStyle: {
                    color: 'black'
                },
            }}>
                <Stack.Screen name="Bottom" component={Bootom} options={{ title: '首页' }} />
                <Stack.Screen name="Detail" component={Detail} screenOptions={{
                    headerLeft: () => null
                }} />
            </Stack.Navigator>
        </NavigationContainer>

    );
}

export default Index;
```

其中NavigationContainer用于管理和维护导航状态和配置即，它维护了当前导航的状态树，并在组件树中提供 `navigation` 和 `route` 对象。这使得你可以在应用的任何部分访问导航功能，而不需要将导航 props 逐层传递。`const Stack = createNativeStackNavigator();` 是堆栈导航器（Stack Navigator），堆栈导航是一种常见的导航模式，用于管理页面之间的堆栈。每当用户导航到一个新页面时，新页面会被推送到堆栈的顶部，当用户返回时，顶部的页面会被弹出。这个导航模式非常适合实现基于页面的流动，例如表单提交后的确认页面、列表点击后的详情页等。



其中Bottom组件页面为：

``` javascript
import { View, Button, Text, TouchableWithoutFeedback } from 'react-native'
import React from 'react'

import { createBottomTabNavigator } from '@react-navigation/bottom-tabs'
import Home from './subPages/home.js'
import DashBoard from './subPages/dash.js'
import News from './subPages/news.js'
import Self from './subPages/self.js'

import Icon from 'react-native-vector-icons/FontAwesome'


const Tab = createBottomTabNavigator()

export default function Index(props) {
    return (
        <Tab.Navigator>
            <Tab.Screen
                name="Home"
                component={Home}
                options={{
                    headerShown: false,
                    tabBarLabel: '首页',
                    tabBarIcon: ({ color, size }) => (
                        <Icon name='home' color={color} szie={size} />
                    )
                }}

            />
            <Tab.Screen
                name="DashBoard"
                component={DashBoard}
                options={{
                    headerShown: false,
                    tabBarLabel: '面板',
                    tabBarIcon: ({ color, size }) => (
                        <Icon name='dashboard' color={color} szie={size} />
                    )
                }}

            />
            <Tab.Screen
                name="News"
                component={News}
                options={{
                    headerShown: false,
                    tabBarLabel: '新闻',
                    tabBarIcon: ({ color, size }) => (
                        <Icon name='newspaper-o' color={color} szie={size} />
                    )
                }}

            />
             <Tab.Screen
                name="Selef"
                component={Self}
                options={{
                    headerShown: false,
                    tabBarLabel: '我的',
                    tabBarIcon: ({ color, size }) => (
                        <Icon name='user' color={color} szie={size} />
                    )
                }}

            />


        </Tab.Navigator>
    )
}
```

`createBottomTabNavigator` 创建一个底部标签导航器，它在屏幕的底部显示一个标签栏。每个标签对应一个不同的屏幕或视图，用户可以通过点击标签来切换视图。如下图所示：

![image-20240721171656406](https://test-1301661941.cos.ap-nanjing.myqcloud.com/image-20240721171656406.png)

首页列表页面：

```javascript
import { FlatList, Image, View, Text, StyleSheet, TouchableWithoutFeedback, Alert } from 'react-native';
import React from 'react';
import { data } from './const.js';

const ds = data.filter(item => item.article_info.cover_image.length > 0);

export default function Home(props) {

    const onPressHandle = (item) => {
        props.navigation.navigate('Detail', { item })
    }

    return (
        <View>
            <FlatList
                data={ds}
                keyExtractor={(item, index) => index.toString()}
                renderItem={({ item }) => <TouchableWithoutFeedback onPress={() => onPressHandle(item.article_info)}>
                    <View style={styles.flex}>

                        <Image style={styles.img} source={{ uri: item.article_info.cover_image }} />
                        <View style={styles.content}>
                            <Text style={styles.title} numberOfLines={1}>{item.article_info.title}</Text>
                            <Text style={styles.contentd} numberOfLines={1}>{'\u00A0\u00A0'}{item.article_info.brief_content}</Text>
                        </View>
                    </View>
                </TouchableWithoutFeedback>}
            >

            </FlatList>
        </View>
    );
}

const styles = StyleSheet.create({

    img: {
        width: 100,
        height: 50,
    },
    title: {
        fontSize: 16,
        fontWeight: "bold",

    },
    flex: {
        width: '100%',
        marginTop: 10,
        display: 'flex',
        flexDirection: 'row',
        gap: 10,
    },
    content: {
        flex: 1,
        gap: 5,
        paddingRight: 10,
        width: '100%'

    },
    contentd: {
        color: '#515767'
    }
});

```

展示结果即为：

![image-20240721172020283](https://test-1301661941.cos.ap-nanjing.myqcloud.com/image-20240721172020283.png)

在 React Native 中，有几种不同的点击组件可以用于处理用户交互和触发事件。每种组件在处理点击事件时有其特定的用途和行为。以下是常用的点击组件及其主要区别：

### 1. **TouchableOpacity**

- **功能**: `TouchableOpacity` 是一个可以被点击的组件，其点击时会显示一个透明度变化的效果，提供视觉反馈。
- **使用场景**: 适用于任何需要用户点击的地方，特别是当你希望在点击时给用户提供视觉反馈时。
- **特点**: 点击时组件的透明度会减少，给用户一种按钮被按下的感觉。

```javascript
import React from 'react';
import { TouchableOpacity, Text, StyleSheet } from 'react-native';

export default function App() {
  return (
    <TouchableOpacity
      style={styles.button}
      onPress={() => console.log('Button pressed')}
    >
      <Text style={styles.text}>Click Me</Text>
    </TouchableOpacity>
  );
}

const styles = StyleSheet.create({
  button: {
    backgroundColor: '#2196F3',
    padding: 10,
    borderRadius: 5,
  },
  text: {
    color: '#fff',
    fontSize: 16,
  },
});
```

### 2. **TouchableHighlight**

- **功能**: `TouchableHighlight` 是一个可以被点击的组件，点击时会显示一个高亮效果。
- **使用场景**: 适用于需要点击反馈且希望使用背景色变化来指示点击状态的场景。
- **特点**: 点击时背景颜色会变暗，并在点击后恢复原始状态。

```javascript

import React from 'react';
import { TouchableHighlight, Text, StyleSheet } from 'react-native';

export default function App() {
  return (
    <TouchableHighlight
      style={styles.button}
      onPress={() => console.log('Button pressed')}
      underlayColor="#DDDDDD" // 点击时的高亮颜色
    >
      <Text style={styles.text}>Click Me</Text>
    </TouchableHighlight>
  );
}

const styles = StyleSheet.create({
  button: {
    backgroundColor: '#2196F3',
    padding: 10,
    borderRadius: 5,
  },
  text: {
    color: '#fff',
    fontSize: 16,
  },
});
```

### 3. **TouchableWithoutFeedback**

- **功能**: `TouchableWithoutFeedback` 是一个可以被点击的组件，不会显示任何点击反馈效果。
- **使用场景**: 适用于需要捕获触摸事件但不需要任何视觉反馈的场景。
- **特点**: 不会显示任何触摸反馈，适合用作触摸事件的捕获器。

```javascript
import React from 'react';
import { TouchableWithoutFeedback, View, Text, StyleSheet } from 'react-native';

export default function App() {
  return (
    <TouchableWithoutFeedback onPress={() => console.log('Button pressed')}>
      <View style={styles.button}>
        <Text style={styles.text}>Click Me</Text>
      </View>
    </TouchableWithoutFeedback>
  );
}

const styles = StyleSheet.create({
  button: {
    backgroundColor: '#2196F3',
    padding: 10,
    borderRadius: 5,
  },
  text: {
    color: '#fff',
    fontSize: 16,
  },
});
```

### 4. **Pressable**

- **功能**: `Pressable` 是一个更灵活的点击组件，允许你在触摸和点击时执行自定义效果。
- **使用场景**: 适用于需要高级交互处理和触摸反馈的场景。
- **特点**: 提供更多的交互处理方式，比如 `onPressIn`、`onPressOut`、`onLongPress`，并且可以配置不同的样式和行为。

```javascript
import React from 'react';
import { Pressable, Text, StyleSheet } from 'react-native';

export default function App() {
  return (
    <Pressable
      style={({ pressed }) => [
        styles.button,
        { backgroundColor: pressed ? '#DDDDDD' : '#2196F3' },
      ]}
      onPress={() => console.log('Button pressed')}
    >
      <Text style={styles.text}>Click Me</Text>
    </Pressable>
  );
}

const styles = StyleSheet.create({
  button: {
    padding: 10,
    borderRadius: 5,
  },
  text: {
    color: '#fff',
    fontSize: 16,
  },
});
```

### 主要区别

- **反馈效果**:
  - **`TouchableOpacity`**: 渐变透明度效果。
  - **`TouchableHighlight`**: 背景色变暗。
  - **`TouchableWithoutFeedback`**: 无视觉反馈。
  - **`Pressable`**: 可以自定义反馈效果，根据状态变化应用不同样式。
- **灵活性**:
  - **`TouchableOpacity`** 和 **`TouchableHighlight`**: 主要用于简单的点击反馈。
  - **`Pressable`**: 提供更多的交互处理和样式配置选项，适合需要复杂反馈的场景。
  - **`TouchableWithoutFeedback`**: 适用于不需要视觉反馈的触摸事件捕获。



路由的跳转为：![image-20240721172700034](https://test-1301661941.cos.ap-nanjing.myqcloud.com/image-20240721172700034.png)

其中第二个参数可以在目标页面通过 props.route.params获取到。
