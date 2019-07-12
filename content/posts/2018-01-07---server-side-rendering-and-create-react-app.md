---
title: create-react-app에 Server Side Rendering을 더한 예제
date: 2018-01-07 23:40:00
template: "post"
draft: false
slug: "/posts/server-side-rendering-and-create-react-app"
category: "Javascript"
tags: 
- javascript
- react
- create-react-app
---
create-react-app은 server-side-rendering을 지원하지 않는다. 지원하지 않는다기 보다 얘는 react앱을 만들기 위해 기본셋팅만 모아둔 제로컨피그 툴이다. 그래서 얘는 빌드하면 js, css파일을 내 뱉을 뿐이다.
여기에 간단하게 NodeJS Server(Koa)를 더 하면 쉽게 서버사이드렌더링을 구현할 수 있다. 물론 이것을 전부 제공 하는 next.js 프레임워크가 있긴 있음.

### Server Side Rendering?
ReactApp은 원래 빈 html div에 스크립트를 로드해서 그리는 건데, 이걸 서버에서 미리 그려줘서 index.html을 내릴 때 그리겠다라는 것. 이걸 하게 되면 아래와 같은 장점이 있다.
1. 검색엔진최적화(SEO)를 할 수 있다. create-react-app은 index.html파일이 정적이기 때문에 url별로 다른 데이터를 내리지 못하는데, 이걸 할 수 있다.
2. 초기 로드를 서버에서 해서 마크업을 먼저 내려주므로 사용자가 스크립트 로드할 동안 빈화면을 볼 시간이 줄어듬.

애초에 React는 SSR을 염두해두고 제작되어있다. react-dom-server 문서 참조.
https://reactjs.org/docs/react-dom-server.html
이 메소드는 단순 초기데이터를 스트링으로 받는 함수다. 이걸 이용해서 서버에서는 로드해서 index.html에 꾸셔넣으면 된다. React16이 되면서 hydrate라는 함수가 생기면서 두 번 로드되어서 그리지 않게 서버에서 그린 걸 재사용하는 함수도 생겼다.

## create-react-app에 ajax로 데이터를 가져와 SSR로 단순 그리는 예제 정리.

### 1. create-react-app 프로젝트 생성
ServiceWorker는 삭제할 것. 안그러면 클라캐시를 먹기이기 때문에 서버사이드렌더링을 할 수 없다. index.html이 캐시가 된다. ServiceWorker 쪽을 좀 더 공부해봐야겠다.
그리고 render메소드를 hydrate로 바꿔야 한다. 이 부분은 서버에서 그린 데이터를 클라에서 재활용하겠다라는 것. 16에서 새로 나온 것인데, 이전 버전은 두 번 렌더되는 형태였을 것.
```shell
npm install -g create-react-app
create-react-app ssr-react
cd ssr-react
```
```javascript
index.js

ReactDOM.hydrate(<App />, document.getElementById('root'));
// registerServiceWorker();
```

### 2. api 서버 생성
koa로 간단한 api 정보를 내려주는 서버를 만든다. koa는 기본적인 게 아무것도 없어서 미들웨어를 하나씩 다 추가해야한다. api에 필요한 미들웨어는 본서버가 호출할 수 있게 cors(타도메인을 호출할꺼라서...), 결과값으로 json을 받을 수 있게 json미들웨어 두 개를 설치해서 만듬.

```shell
mkdir simple-api
cd simple-api
npm init
npm install koa koa-json @koa/cors --save
```
```javascript
index.js
const Koa = require('koa');
const cors = require('@koa/cors');
const json = require('koa-json');
const app = new Koa();

app.use(cors());
app.use(json());
app.use(async ctx => {
  ctx.body = ['apple', 'banana', 'strawberry', 'orange'];
});

app.listen(3002);
```
```shell
# 실행
node index.js
```
3001번서버에서 3002번api를 호출하는 구조.


### 2. unfetch를 이용해서 단순 데이터가져오는 로직 추가.
데이터가져오기 위한 라이브러리 필요. isomorphic-unfetch가 적당함. 왜냐하면 isomorphic-unfetch는 nodejs 콜도 지원하기 때문. SSR을 하기 위해서는 nodejs와 client를 둘 다 지원하는 것이 필요함.
```shell
npm install isomorphic-unfetch --save
```
```javascript
App.js
import React, { Component } from 'react';
import logo from './logo.svg';
import './App.css';
import fetch from 'unfetch';

class App extends Component {
  static async getInitialState() {
    const res = await fetch(`http://localhost:3001/api/fruits`);
    return await res.json();
  }

  constructor() {
    super();
    this.state = {data: []};
  }

  async componentWillMount() {
    const data = await App.getInitialState();
    this.setState({data});
  }

  render() {
    const {data} = this.state;
    return (
      <div className="App">
        <header className="App-header">
          <img src={logo} className="App-logo" alt="logo" />
          <h1 className="App-title">Welcome to React</h1>
        </header>
        <p className="App-intro">
          To get started, edit <code>src/App.js</code> and save to reload.
        </p>
        <ul>
          {data.map((d, i) => (<li key={`fruits-${i}`}>{d}</li>))}
        </ul>
      </div>
    );
  }
}

export default App;
```
getInitalState라는 static메소드를 정의하고 데이터를 가져오는 단순 React로직.

{% asset_img image1.png 데모화면 %}

### 3. index.html, js, css 등을 내려줄 본서버 생성
이것도 코아로 생성을...이건 create-react-app 생성한 폴더에서 한다.
```shell
npm install koa koa-static 
```
```javascript
server.js
import serve from 'koa-static';
import path from 'path';
import Koa from 'koa';
import {promisify} from 'util';
import fs from 'fs';
import App from "./App";
import React from 'react';
import {renderToString} from 'react-dom/server';

const app = new Koa();
const readFile = promisify(fs.readFile);

app.use(serve(path.join(__dirname, '../build'), {index: 'nothing.html'}));
app.use(async ctx => {
  const htmlData = await readFile(path.resolve(__dirname, '..', 'build', 'index.html'), 'utf8');
  const initialState = await App.getInitialState();
  const markup = renderToString(<App data={initialState}/>);
  console.log('markup', markup);
  ctx.body = htmlData
    .replace(`<div id="root"></div>`, `<div id="root">${markup}</div>`)
    .replace(`INITIAL_STATE=""`, `INITIAL_STATE=${initialState ? JSON.stringify(initialState) : `""`}`)
});

app.listen(3002);
```
코드를 보면 App.js에 있는 initialState메소드를 그대로 호출한다. 그래서 그 값을 props에 넣는다. 그래서 renderToString을 통해 markup을 받은 걸 index.html안에 넣는 코드다. 
대충 이런 식으로 하면 초기데이터 기반으로 meta태그도 수정할 수 있게 된다.

### 4. App.js를 서버렌더링에 맞게 수정.
App.js를 수정하지 않으면 데이터 로딩이 두 번 일어 날 것. 그래서 맞게 수정이 필요. 그리고 스타일 관련된 것은 제거. 그러지 않으면 서버에서 못읽기 때문에 에러 발생. 그래서 분기를 하거나 index.js에 css관련된 것을 한 번에 넣어서 처리하는 방법이 있음. ignore-styles라는 라이브러리도 있음.
```javascript
App.js
import React, { Component } from 'react';
import fetch from 'isomorphic-unfetch';

class App extends Component {
  static async getInitialState() {
    const res = await fetch(`http://localhost:3001/api/fruits`);
    return await res.json();
  }

  constructor(props) {
    super(props);
    console.log('props', props);
    // 서버인 경우 data가 들어옴.
    const {data} = props;
    // 클라인 경우 initialState값이 있는지 체크.
    const initialData = typeof window !== 'undefined' && window.INITIAL_STATE ? window.INITIAL_STATE : null;
    // data가 있으면 우선, 그 다음은 initialData.
    this.state = {data: data ? data : (initialData ? initialData : null)};
  }

  async componentWillMount() {
    const {data} = this.props;
    // 데이터가 없는 경우만 로드해서 중복 로드 방지.
    if (!data) {
      this.setState({data: await App.getInitialState()});
    }
  }

  render() {
    const {data} = this.state;
    return (
      <div className="App">
        <header className="App-header">
          <h1 className="App-title">Welcome to React</h1>
        </header>
        <p className="App-intro">
          To get started, edit <code>src/App.js</code> and save to reload.
        </p>
        <ul>
          {data && data.map((d, i) => (<li key={`fruits-${i}`}>{d}</li>))}
        </ul>
      </div>
    );
  }
}

export default App;
```
고친 부분은 constructor와 componentWillMount부분. 서버에서 내려준 경우는 data가 props으로 있을 것이고, 클라에서 확인할 때에는 INITIAL_STATE가 있을 것. 그걸 기반으로 state값을 정하는 코드가 추가되었음. 그러면 아마 새로고침할 때 밑에 리스트가 느리게 그려지는 게 아니라 한 번에 보이게 됨.

### 5. 실행은 바벨로...
현재 server.js코드는 react형태로 작성이 되어 있는데, 이것은 node.js에서 인식하지 못함. 그래서 babel-node로 실행을 해봐야함.
실서버에 배포할 때에는 babel로 컴파일 후 배포해야함.
```shell
npm install babel-cli --save-dev
```

실행 스크립트에 server:start 추가.
```javascript
package.json
"scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test --env=jsdom",
    "eject": "react-scripts eject",
    "server:start": "NODE_ENV=development babel-node src/server --presets env,react-app"
  }
```
```shell
npm run build
npm run server:start
```
그러면 소스보기를 했을 때 서버렌더링이 된 것을 볼 수 있다.
```html
<div id="root"><div class="App" data-reactroot=""><header class="App-header"><h1 class="App-title">Welcome to React</h1></header><p class="App-intro">To get started, edit <code>src/App.js</code> and save to reload.</p><ul><li>apple</li><li>banana</li><li>strawberry</li><li>orange</li></ul></div></div>
```

리얼에선 babel 컴파일해서 써야함. node-babel은 런타임에 아마 일을 하기 때문에...
```shell
# 대충 이런식으로...
NODE_ENV=production babel src -d dist --presets env,react-app
NODE_ENV=production node dist/server.js
```