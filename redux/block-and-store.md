# Redux

## Архитектура блоков

В целях разделения логики блока и увеличения "полезной нагрузки" всю работу блока со `store` (редьюсеры, экшены) следует хранить в элементе `store`.

Например:

`auth/__store/auth__store.js`

## Почему именно так

Например есть:
блок `A`
блок `B`

Блок `A` использует блок`B`. Блок `B` генерирует какие-то события внутри себя, которые изменяют хранилище (которое используют блоки `A` и `B`). 
Но блок `B` не может подключить модуль, т.к. будет циклическая зависимость.

#### Решение 1. 

Дублирования кода. Писать повторящиеся `dispatch` и не забывать про префиксы `action`.

#### Решение 2. 

В блоке `B` генерировать`_emit()`. 
В блоке `A` подписываться через `this.findChildBlock('B').on()` (но блок `B` может ведь быть не только внутри...) и изменять `store` (а это как раз противоречит философии Redux о потоке данных)

#### Решение 3. 

Блок `A` при разных событиях дёргает свой `store`.
Блок `B` при разных событиях дёргает `stote` блока `А` (+ другие `store`, если надо).

---------------------

В данном случае 2-й вариант отметается. Остаются 1 и 3. 

- Вариант 1 - много излишков
- Вариант 3 - более чистая архитектура


## Пример auth__store

```js
modules.define('auth__store', ['store', 'server-store'],
(provide, store, server) => {
  const SET_TOKEN = 'AUTH/SET_TOKEN';
  const SET_USER_DATA = 'AUTH/SET_USER_DATA';
  const DESTROY = 'AUTH/DESTROY';

  /**
   * Редьюсер для аутентификации пользователя
   * @param  {object} state - prevent state
   * @param  {object} action - action
   * @return {object} new auth state
   */
  function authReducer(state = {}, action) {
    switch (action.type) {
      case SET_TOKEN:
        return Object.assign({}, state, action.payload);
      case SET_USER_DATA:
        return Object.assign({}, state, action.payload);
      case DESTROY:
        return {};
      default:
        return state;
    }
  }

  store.injectAsyncReducer('auth', authReducer);

  /**
   * @exports API
   */
  provide({
    login: data => {
      server.load('userToken', {body: data})
      .then(tokenData => {
        store.dispatch({type: SET_TOKEN, payload: {token: tokenData.token}});
        return server.load('userData', {token: tokenData.token});
      })
      .then(userData => {
        store.dispatch({type: SET_USER_DATA, payload: {user: userData}});
      })
      .catch(err => {
        console.error(err);
      });
    },

    logout: () => {
      store.dispatch({type: DESTROY});
    },

    isAuth: () => {
      const state = store.getState();
      if (state.auth && state.auth.user) {
        return true;
      }
      return false;
    }
  });
});

```
