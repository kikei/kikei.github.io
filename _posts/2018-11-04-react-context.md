---
layout: post
title:  "React Contextを少し使ってみた"
categories: ai
---

React 16.3 より React Context という機能が正式版でリリースされたらしいので
試してみた。

### 1. React Context について

調べてみたところ、React の "prop drilling" 問題が華麗に解決されるようだ。
Context を利用することで、上層で定義されたプロパティを深くネストした要素まで
簡単に届けることができるようになった。

これまで React+Redux を使う必要があった機能の一部が、本家でも解決できるようになった
という感じのようだ。

詳しくは参考 URI を見てほしい。このページでは解説しない。

使ってみた感じとしては、
React+Redux は Flux パターンがあるので React 単独と比べると依然強力だが、
そこまで大規模でないプロジェクトならば React Context を使うだけで十分かもしれない。

### 2. プロジェクト作成

何を作るかも考えないままに、とりあえずプロジェクト作成。

```shell
$ npm init react-app client
$ cd client
$ npm install react-router-dom
```

とりあえず開発用サーバー起動。

```shell
$ npm start
Compiled successfully!

You can now view client in the browser.

  Local:            http://localhost:3000/
  On Your Network:  http://192.168.0.8:3000/

Note that the development build is not optimized.
To create a production build, use npm run build.
```

簡単。

### 3. 作ったもの

カウンターとメモリスト機能を持った何に使うのかわからないアプリ。

なぜかシンプルモードとアドバンスモードを選べるモード設定機能がある。

シンプルモードではカウンター、メモリストともに最低限の機能しか使えないが、
アドバンスモードに設定すると、もう少し高度な機能が利用可能になる。

#### SettingContext

React Context として用意した Context である。
この Context でモード設定を保持する。

Context はトップレベルで生成されるが、
カウンターとメモリストにも渡されて参照が可能である。

#### Routing

カウンターとメモリストには個別に URI が設定され、Route を使って分離する。

#### ファイル構成

練習なのでほぼ全部 App.js に書いた。

```
$ tree
.
├── App.js
├── index.js
├── serviceWorker.js
└── webpack.config.js
```

### 3. Context 生成

以下はしばらく App.js。

まずライブラリのロード。

```javascript
import React, { Component } from 'react';
import { Switch, Route, Link } from 'react-router-dom';

const SettingContext = React.createContext();
```

Context 生成。

```
const SettingContext = React.createContext();
```

Context のデフォルト状態を `createContext` の引数で指定することもできるらしい。

### 4. トップレベルの作成

モード設定、カウンター、メモリストの各画面を以下のようにルーティングした。

| Path     | Component | View      | Function                                  |
|:---------|:----------|:----------|:------------------------------------------|
| /        | Home      | モード設定 | モード設定を変更できる。                    |
| /counter | Counter   | カウンター | カウンターを表示する。モード設定を参照する。 |
| /memo    | Memo      | メモリスト | メモリストを表示する。モード設定を参照する。 |

上記のコンポーネントへモード設定を引き渡すため、
`<Switch/>` ごと `SettingContext.Provider` で囲った。

```javascript
class App extends Component {
  constructor(props) {
    super(props)
    this.state = {
      setting: {
        mode: 'simple',
        changeMode: this.changeMode.bind(this),
        isSimpleMode: () => this.state.setting.mode === 'simple'
      }
    }
  }
  changeMode(e) {
    const setting = Object.assign(this.state.setting, {mode: e.target.value})
    this.setState({setting: setting})
  }
  render() {
    return (
      <div className="App">
        <nav>
          <Link to="/">Home</Link> /
          <Link to="/counter">Counter</Link> /
          <Link to="/memo">Memo</Link>
        </nav>
        <SettingContext.Provider value={this.state.setting}>
          <Switch>
            <Route exact path="/" component={Home}/>
            <Route path="/counter" component={Counter}/>
            <Route path="/memo" component={Memo}/>
          </Switch>
        </SettingContext.Provider>
      </div>
    );
  }
}
```

### 5. モード設定画面

さっき `SettingContext.Provider` で渡した値を、
`SettingContext.Consumer` で受け取れる。

```javascript
const Home = () => (
    <SettingContext.Consumer>
    {
      (setting) => {
        return (
          <div>
            <div>Select mode:</div>
            <div>
                  <label>
                    Simple:
                    <input type="radio" name="mode" value="simple"
                           checked={setting.isSimpleMode()}
                           onChange={setting.changeMode}/>
                  </label>
                </div>
                <div>
                  <label>
                    Advanced:
                    <input type="radio" name="mode" value="advanced"
                           checked={!setting.isSimpleMode()}
                           onChange={setting.changeMode}/>
                  </label>
                </div>
              </div>
        )
      }
    }
  </SettingContext.Consumer>
)
```

![モード設定画面](/images/screenshots/2018-11-04-app-home.png)

*図5.1. モード設定画面*

### 6. カウンター画面の作成

基本的な機能は自分で持つ。

ただしモード設定は Context で受け取る。
`SettingContext.Consumer` を使うだけ。

```javascript
class Counter extends Component {
  constructor(props) {
    super(props)
    this.increment = this.increment.bind(this)
    this.decrement = this.decrement.bind(this)
    this.changeStepInput = this.changeStepInput.bind(this)
    this.state = {
      count: 0,
      // Advanced mode
      step: 1
    }
  }
  increment() {
    this.setState({count: this.state.count + this.state.step})
  }
  decrement() {
    this.setState({count: this.state.count - this.state.step})
  }
  changeStepInput(e) {
    const step = parseInt(e.target.value)
    this.setState({step: step})
  }
  render() {
    const state = this.state
    return (
      <SettingContext.Consumer>
        {
          (setting) => {
            return (
              <React.Fragment>
                <div>
                  <button onClick={this.increment}>incr</button>
                  <button onClick={this.decrement}>decr</button>
                </div>
                {
                  !setting.isSimpleMode() && 
                  <div>
                    Step:
                    <input type="text" onChange={this.changeStepInput}/>
                  </div>
                }
                <div>Count: {state.count}</div>
              </React.Fragment>
            )
          }
        }
      </SettingContext.Consumer>
    )
  }
}
```

![カウンター画面](/images/screenshots/2018-11-04-app-counter.png)

*図6.1. カウンター画面*

### 7. メモリスト画面の作成

```javascript
class Memo extends Component {
  constructor(props) {
    super(props)
    this.changeTextInput = this.changeTextInput.bind(this)
    this.submitText = this.submitText.bind(this)
    this.state = {
      memos: [],
      textInput: "",
    }
    // Advanced
    this.clearAll = this.clearAll.bind(this)
  }
  changeTextInput(e) {
    this.setState({
      textInput: e.target.value
    })
  }
  submitText(e) {
    this.setState({
      memos: this.state.memos.concat([this.state.textInput]),
      textInput: ''
    })
  }
  clearAll(e) {
    this.setState({
      memos: []
    })
  }
  render() {
    const state = this.state
    return (
      <SettingContext.Consumer>
        {
          (setting) => {
            return (
              <React.Fragment>
                <input type="text" size="30" value={state.textInput}
                       onChange={this.changeTextInput}
                       placeholder="Write your memo here"/>
                <button onClick={this.submitText}>Add</button>
                {
                  !setting.isSimpleMode() &&
                  <button onClick={this.clearAll}>Clear all</button>
                }
                <div>Memos:
                  <ol>
                    {
                      state.memos.map((memo, i) => <li key={i}>{memo}</li>)
                    }
                  </ol>
                </div>
              </React.Fragment>
            )
          }
        }
      </SettingContext.Consumer>
    )
  }
}
```

![メモリスト画面](/images/screenshots/2018-11-04-app-memo.png)

*図7.1. メモリスト画面*

### 8. エントリポイントの修正

ここは index.js に書く。

プロジェクト作成時にデフォルトで生成された `<App/>` に対し、
`<BrowserRouter/>` で囲んだだけ。

```js
import React from 'react';
import ReactDOM from 'react-dom';
import App from './App';
import * as serviceWorker from './serviceWorker';
import { BrowserRouter } from 'react-router-dom';

ReactDOM.render(
  <BrowserRouter>
    <App />
  </BrowserRouter>, 
  document.getElementById('root'));

// If you want your app to work offline and load faster, you can change
// unregister() to register() below. Note this comes with some pitfalls.
// Learn more about service workers: http://bit.ly/CRA-PWA
serviceWorker.unregister();
```

### 9. 参考

- [Context – React](https://reactjs.org/docs/context.html)
- [How to build your own React\-Router with new React Context Api](https://medium.com/@stevenkoch/how-to-build-your-own-react-router-with-new-react-context-api-1647406b9b93)
- [Redux vs\. React Context API / Habr \- Techort](http://www.techort.com/redux-vs-react-context-api-habr/)
- [React Context APIについて - Qiita](https://qiita.com/gipcompany/items/f867bc7e9241f2387ce9)
- [reactjs \- How to use context api with react router v4? \- Stack Overflow](https://stackoverflow.com/questions/50155909/how-to-use-context-api-with-react-router-v4)
- [reactjs \- React Router Switch and exact path \- Stack Overflow](https://stackoverflow.com/questions/51961135/react-router-switch-and-exact-path)
- [React Router v4: The Complete Guide — SitePoint](https://www.sitepoint.com/react-router-v4-complete-guide/)
