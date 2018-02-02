# normalizr / reselect

## はじめに

redux を使っているプロジェクトは結構あるが、store を見てみるとネストが深かったり reducer での処理が辛かったりとしっかり設計されてもるものは少ない。store も DB と同じように設計することで参照しやすくなったり複雑さを回避できる。今回は normalizr で store を正規化して reselect でキレイにデータを取り出してみる。

## [normalizr](https://github.com/paularmstrong/normalizr) (json を正規化するライブラリ)

よくある api response

```json
{
  "id": "123",
  "author": {
    "id": "1",
    "name": "Paul"
  },
  "title": "My awesome blog post",
  "comments": [
    {
      "id": "324",
      "commenter": {
        "id": "2",
        "name": "Nicole"
      }
    }
  ]
}
```

そのまま store に保存すると reducer で死ぬ

例えば commenter の名前に更新があった場合

```javascript
export default (state = {}, action) => {
  switch (action.type) {
    case "UPDATE_COMMENTER_NAME":
      return {
        ...state,
        comments: state.comments.map(comment => {
          if (comment.commenter.id === action.payload.id) {
            return {
              id: comment.id,
              commenter: {
                id: action.payload.id,
                name: action.payload.name // ここが更新
              }
            };
          }
          return comment;
        })
      };
    default:
      return state;
  }
};
```

たぶん、こう。これであっているかどうかを確認するのも大変...

normalizr を使えば api response を正規化して store のネストを少なくすることができる。

```javascript
{
  result: "123",
  entities: {
    "articles": {
      "123": {
        id: "123",
        author: "1",
        title: "My awesome blog post",
        comments: ["324"]
      }
    },
    "users": {
      "1": { "id": "1", "name": "Paul" },
      "2": { "id": "2", "name": "Nicole" }
    },
    "comments": {
      "324": { id: "324", "commenter": "2" }
    }
  }
}
```

参照用 id(result) とデータの実体 (entities) に分離される。

articles, users, comments を分けて store に格納する。

結果、UPDATE_COMMENTER_NAME の処理が簡素になる

```javascript
// users だけの reducer
export default (state = {}, action) => {
  switch (action.type) {
    case "UPDATE_COMMENTER_NAME":
      const { id, name } = action.payload;
      return {
        ...state,
        [id]: { id, name }
      };
    default:
      return state;
  }
};
```

どっちがいいかは一目瞭然。保守性が爆上がりする。

本当は api を返す段階で正規化されていることが望ましいが、実際のプロジェクトはそうもいかない場面が多いので、normalizr を使ってクライアント側で正規化して頑張ろう。

## [reselect](https://github.com/reactjs/reselect) (redux 用のセレクターライブラリ)

redux のセレクターとは store から必要な値を取り出すもの。store は 1 つのツリーオブジェクト。view で使うのはツリーの中の一部だけで不必要なデータはなるべく渡したくない。

redux でよくあるセレクター

```javascript
const mapStateToProps = state => {
  const someData = state.some.data;
  const filteredData = expensiveFiltering(someData);
  const sortedData = expensiveSorting(filteredData);

  return { data: sortedData };
};
```

ダメな点

* some.data にアクセスする components が他にもあった場合に同じ記述をいくつも書かないといけない
* some.data の構造に変更があった場合いろんなところで書き換えないといけない
* state.some.data とは関係ない state の更新であった場合にも filter, sort 処理が走る

reselect を使って書き換え

```javascript
const selectSomeData = state => state.some.data;

// prettier-ignore
const selectFilteredSortedData = createSelector(
  selectSomeData,
  someData => {
    const filteredData = expensiveFiltering(someData);
    const sortedData = expensiveSorting(filteredData);
    return sortedData;
  }
);

const mapStateToProps = state => {
  const filteredSortedData = selectFilteredSortedData(state);
  return { data: filteredSortedData };
};
```

5 行目: inputSelector

6-10 行目: outputSelector

inputSelector が以前の state と異なる場合に outputSelector が実行される

inputSelector が以前の state と同じ場合は outputSelector が実行されず前回のキャッシュを返す(Memoized と呼ばれる)

さっきのダメな点がすべて解決される。無駄なレンダリングも防げるのでオススメ。

## おわりに

normalizr と reselect の組み合わせは非常に強力。正規化によって store からのデータ抽出も楽になる。normalizr と reselect は実は redux 公式でも言及されている。積極的に使っていこう。
