# firebase-ec-demo
FirebaseでECっぽいサービスをFirestoreで構築するときのモデルとSecurityRuleのサンプルです。  

## お詫び
実はクライアント側の実装もして実際に読み書きを試せるようにしたかったのですが時間足らずでモデルの構成とfirestore.rulesの公開だけになりました :pray:  
こんな感じで書くんだよっていう雰囲気が伝われば幸いです。

## おことわり
- 実際のサービスで使用されている構成とは異なります
- あくまでも書き方のサンプルなので、動作の完全な保証やセキュリティの担保まではできません。必ず動作やセキュリティチェックは行いましよう。

# 概要
Cloud Firestoreを使ってECっぽいサービスのDBを設計する場合のモデル定義とセキュリティルールのサンプルになります。  
rulesをどう書いたら良いんだろうというののサンプルなので、実際に運用する場合と比べて設計はかなり簡素にして、  
一部省略しているモデルもあります（注文管理、売上管理、口座情報等）

## おおまかな流れ
アプリケーションの中で、販売者(以降Sellerとする)、購入者(Customerとする)の2つ、ユーザー属性がある想定で行きます。

### Seller
Sellerは、Firebase Authenticationにより定められた認証方法で認証した後、auth.uidを基に、sellerモデルを生成して保存します。  
また、この時Sellerに紐づくshopモデルも同時に作成します。

Sellerは商品をproductモデルに必要な情報を書き込んで出品することができます。productは公開/非公開の設定が可能です。

### Customer
Customerは同じくAuthenticationにより認証した後、auth.uidを基にcustomerモデルを作成して保存します。  
また、この時Customerに紐づくcartモデルも同時に作成します。  

Customerは公開になっている商品を検索等おこない、商品の詳細を見たあと、購入したいと思った商品をcartモデルに商品を追加することができます。  
購入したい商品を追加するときは、数量も指定することができます。

## document, rulesの基本方針
- 全てのドキュメントには`createdAt`, `updatedAt`のフィールドをもたせるようにしています
- 基本的にはdeleteのoperationは用いず、`isActive`というフラグでドキュメントが有効なものか、論理削除されたものかをチェックするようにしています

- 基本的には全てのドキュメントの読み書きに関してFirebaseの認証が通っている状態を想定しています
- ドキュメントをcreateするときは、必要なFieldが揃っているかチェックします
- 書き込むときのFieldが正しい型や値になっているか可能な限りチェックします
- customerとcartのドキュメントのように、バッジ処理を用いて同時に作成する場合は、`getAfter`関数を用いて同時に書き込まれているか保証するようにしています


# 使用するモデル一覧

- seller
- customer
- shop
- cart
- cart/product
- product

## seller

- path: `/seller/{sellerID}`
  - `sellerID`はAuthのuidと一致するようにする

```js
{
  seller: {
    name: string,
    shopRef: reference,
    isActive: bool,
    createdAt: timestamp,
    updatedAt: timestamp
  }
}

```


key | type | desc
--- | --- | ---
name | string | ユーザー名
shopRef | reference | ショップのDocumentReference
isActive | bool | 論理削除フラグ。falseで論理削除
createdAt | timestamp | Documentの作成日時
updatedAt | timestamp | Documentの更新日時

## customer

- path: `/customer/{customerID}` 
  - `customerID`はAuthのuidと一致するようにする

```js
{
  customer: {
    name: string,
    cartRef: reference,
    isActive: bool,
    createdAt: timestamp,
    updatedAt: timestamp
  }
}
```


key | type | desc
--- | --- | ---
name | string | ユーザー名
cartRef | reference | カートのDocumentReference
isActive | bool | 論理削除フラグ。falseで論理削除
createdAt | timestamp | Documentの作成日時
updatedAt | timestamp | Documentの更新日時


### securecustomer

- path: `/securecustomer/{customerID}`
  - ※`customerID`は`/customer/{customerID}`と同一のIDとなるようにする

```js
{
  securecustomer: {
    gender: string,
    birthday: timestamp,
    paymentServiceCustomerID: string,
    createdAt: timestamp,
    updatedAt: timestamp
  }
}
```

key | type | desc
--- | --- | ---
gender | string | 性別。 'male', 'female', 'unknown' のいずれか
birthday | timestamp | ユーザーの誕生日
paymentServiceCustomerID | string | 決済サービスで用いるID
createdAt | timestamp | Documentの作成日時
updatedAt | timestamp | Documentの更新日時

## shop

- path: `/shop/{shopID}` 

```js
{
  shop: {
    name: string,
    sellerRef: reference,
    isActive: bool,
    createdAt: timestamp,
    updatedAt: timestamp
  }
}
```

key | type | desc
--- | --- | ---
name | string | ショップ名
sellerRef | reference | 販売者ユーザーののDocumentReference
isActive | bool | 論理削除フラグ。falseで論理削除
createdAt | timestamp | Documentの作成日時
updatedAt | timestamp | Documentの更新日時

### secureshop

- path: `/secureshop/{shopID}`
  - ※`shopID`は`/secureshop/{shopID}`と同一のIDとなるようにする

```js
{
  secureshop: {
    paymentServiceAccountID: string,
    createdAt: timestamp,
    updatedAt: timestamp
  }
}
```

key | type | desc
--- | --- | ---
paymentServiceAccountID | string | 決済サービスで用いるID
createdAt | timestamp | Documentの作成日時
updatedAt | timestamp | Documentの更新日時

## cart

- path: `/cart/{cartID}` 

```js
{
  cart: {
    customerRef: reference,
    createdAt: timestamp,
    updatedAt: timestamp
  }
}
```

key | type | desc
--- | --- | ---
name | string | ショップ名
customerRef | reference | 購入者ユーザーののDocumentReference
isActive | bool | 論理削除フラグ。falseで論理削除
createdAt | timestamp | Documentの作成日時
updatedAt | timestamp | Documentの更新日時

### cart/products
SubCollection of `cart`.

- path: `/cart/{cardID}/products/{productID}` 

```js
{
  cart/products: {
    quantity: number,
    cartRef: reference,
    productRef: reference,
    createdAt: timestamp,
    updatedAt: timestamp
  }
}
```

key | type | desc
--- | --- | ---
quantity | number | 購入する数量
cartRef | reference | cartのDocumentReference
productRef | reference | productのDocumentReference
createdAt | timestamp | Documentの作成日時
updatedAt | timestamp | Documentの更新日時

## product

- path: `/product/{productID}`

```js
{
  product: {
    name: string,
    desc: string,
    stock: number,
    price: number,
    postage: number,
    estimatedShippingDays: number,
    isPublished: bool,
    isActive: bool,
    shopRef: reference,
    createdAt: timestamp,
    updatedAt: timestamp
  }
}
```

key | type | desc
--- | --- | ---
name | string | 商品名
desc | string | 商品の説明
stock | number | 在庫数
price | number | 商品の値段
postage | number | 送料
estimatedShippingDays | number | 商品発送までの目安日数
isPublished | bool | 商品が公開状態かどうか
isActive | bool | 論理削除フラグ。falseで論理削除
shopRef | reference | 商品を出品するshopのDocumentReference
createdAt | timestamp | Documentの作成日時
updatedAt | timestamp | Documentの更新日時
