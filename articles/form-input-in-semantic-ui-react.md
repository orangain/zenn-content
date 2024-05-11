---
title: "Semantic UI ReactのForm.Inputとはなんなのか"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
---

これは 2020-07-05 に書かれた記事を再投稿したものです。2024年現在での正確性は保証できません。

[Semantic UI React](https://react.semantic-ui.com/) を使っていて `Input` と `Form.Input` の違いがよくわかっていなかったので、ちゃんと調べてみました。

バージョンは次のとおりです。

* React: 16.13.1
* Semantic UI React: 0.88.2

## はじめに

[リファレンス](https://react.semantic-ui.com/collections/form/) で `Form.Input` を選択すると書かれている通り、 `Form.Input` は `<Form.Field control={Input} />` のシンタックスシュガーです。

> Sugar for `<Form.Field control={Input} />`.

さらに `From.Field` の項目を読むと、次のように書かれています。

> A field is a form element containing a label and an input.

しかし、バリデーションエラー用のLabelも重要な役割を担っているので、 `From.Field` は次の3つを含む複合コンポーネントだと捉えた方が良いでしょう [^1] 。

* 項目名を表すlabel
* メインの入力欄であるInput
* バリデーションエラー用のLabel

さて、 `Form.Field` の中身を記述する方法は、大きく分けて次の2つがあります。

* `control` を指定する方法（ `Form.Input` を使ったときに発生）
* `children` （子要素）を指定する方法

個人的には後者の方が暗黙的な挙動が少なくて好きですが、前者の方が記述は楽です。
この記事では、同じDOM構造を実現するための2種類の書き方を比較することで、  `Form.Input` を使った場合に行われることを理解します。

## 2種類の記述方法を比較する

ここでは次の2つの記述方法でほぼ同じDOM構造を実現します。

1. `Form.Input` のみを使う
2. `Form.Field` の子要素に `Input` を含める


ソースコードは次の通りです。

```js
import React, { useState } from "react";
import { Form, Input, Label } from "semantic-ui-react";

const FormExampleFieldError = () => {
  const [firstName, setFirstName] = useState("");
  const handleChange = e => {
    setFirstName(e.target.value);
  };
  const error = firstName === "" ? "First name is required" : undefined;

  return (
    <Form>
      {/* 1. Form.Inputのみを使う */}
      <Form.Input
        type="text"
        name="firstName"
        id="input1-firstName"
        placeholder="First name"
        value={firstName}
        onChange={handleChange}
        label="First name"
        error={error}
      />

      {/* 2. Form.Fieldの子要素にInputを含める */}
      <Form.Field error={!!error}>
        <label htmlFor="input2-firstName">First name</label>
        <Input
          type="text"
          name="firstName"
          id="input2-firstName"
          placeholder="First name"
          value={firstName}
          onChange={handleChange}
          aria-describedby={error && "input2-firstName-error-message"}
          aria-invalid={!!error}
        />
        {error && (
          <Label
            prompt
            pointing="above"
            id="input2-firstName-error-message"
            role="alert"
            aria-atomic
          >
            {error}
          </Label>
        )}
      </Form.Field>
    </Form>
  );
};

export default FormExampleFieldError;
```

### レンダリング結果: エラー無しの場合

エラーが無い場合、レンダリング結果は次のようになりました。同等のDOM構造になっていることがわかります。なお `aria-invalid="false"` はデフォルト値なので、属性が無いのと同じ意味です。

```html
<form class="ui form">
    <!-- 1. Form.Inputのみを使う -->
    <div class="field"><label for="input1-firstName">First name</label>
        <div class="ui input"><input name="firstName" placeholder="First name" id="input1-firstName" type="text"
                value="a"></div>
    </div>
    
    <!-- 2. Form.Fieldの子要素にInputを含める -->
    <div class="field"><label for="input2-firstName">First name</label>
        <div class="ui input"><input name="firstName" id="input2-firstName" placeholder="First name"
                aria-invalid="false" type="text" value="a"></div>
    </div>
</form>
```

### レンダリング結果: エラー有りの場合

エラーが有る場合、レンダリング結果は次のようになりました。こちらも同等のDOM構造になっています。 `Form.Input` を使うと、 `aria-invalid` や `aria-describedby` などのアクセシビリティ関連の属性が自動で追加されていることがわかります。

```html
<form class="ui form">
    <!-- 1. Form.Inputのみを使う -->
    <div class="error field"><label for="input1-firstName">First name</label>
        <div class="ui input"><input aria-describedby="input1-firstName-error-message" aria-invalid="true"
                name="firstName" placeholder="First name" id="input1-firstName" type="text" value=""></div>
        <div class="ui pointing above prompt label" id="input1-firstName-error-message" role="alert" aria-atomic="true">
            First name is required</div>
    </div>
    
    <!-- 2. Form.Fieldの子要素にInputを含める -->
    <div class="error field"><label for="input2-firstName">First name</label>
        <div class="ui input"><input name="firstName" id="input2-firstName" placeholder="First name"
                aria-describedby="input2-firstName-error-message" aria-invalid="true" type="text" value=""></div>
        <div id="input2-firstName-error-message" role="alert" aria-atomic="true" class="ui pointing above prompt label">
            First name is required</div>
    </div>
</form>
```

## まとめ

Semantic UI Reactでは、 `Form.Input` を使うことで、少ない記述で次の3要素を持つ入力欄を作成できます。

* 項目名を表すlabel
* メインの入力欄であるInput
* バリデーションエラー用のLabel

特にバリデーションエラーの表示は自前で記述するとそれなりに大変なので、積極的に使っていきたいです。

## 参考

* [Form.Fieldのソースコード](https://github.com/Semantic-Org/Semantic-UI-React/blob/v0.88.2/src/collections/Form/FormField.js)

## おまけ: 子要素にinputを含めた場合

[リファレンス](https://react.semantic-ui.com/collections/form/) に書かれているように、 `Form.Field` には `Input` の代わりに、HTMLの `input` 要素を含めることもできます。この場合、若干異なるDOM構造が生成されるものの、見た目は同じになりました。DOM構造の差異は、inputが `<div class="ui input">` によって囲まれない点だけです。

ソース

```js
      {/* 3. Form.Fieldの子要素にhtmlのinputを含める */}
      <Form.Field error={error}>
        <label htmlFor="input3-firstName">First name</label>
        <input
          type="text"
          name="firstName"
          id="input3-firstName"
          placeholder="First name"
          value={firstName}
          onChange={handleChange}
          aria-describedby={error && "input3-firstName-error-message"}
          aria-invalid={!!error}
        />
        {error && (
          <Label
            prompt
            pointing="above"
            id="input3-firstName-error-message"
            role="alert"
            aria-atomic
          >
            {error}
          </Label>
        )}
      </Form.Field>
```

レンダリング結果: エラー無しの場合

```html
    <!-- 3. Form.Fieldの子要素にhtmlのinputを含める -->
    <div class="field"><label for="input3-firstName">First name</label><input type="text" name="firstName"
            id="input3-firstName" placeholder="First name" aria-invalid="false" value="a"></div>
```

レンダリング結果: エラー有りの場合

```html
    <!-- 3. Form.Fieldの子要素にhtmlのinputを含める -->
    <div class="error field"><label for="input3-firstName">First name</label><input type="text" name="firstName"
            id="input3-firstName" placeholder="First name" aria-describedby="input3-firstName-error-message"
            aria-invalid="true" value="">
        <div id="input3-firstName-error-message" role="alert" aria-atomic="true" class="ui pointing above prompt label">
            First name is required</div>
    </div>
```

エラーの有無にかかわらず、見た目では区別がつきません。

![エラーが無い場合の見た目](https://crieit.now.sh/upload_images/5a11f6f3844392eef8bb653508a390ab5f01e4095de34.png)

![エラーが有る場合の見た目](https://crieit.now.sh/upload_images/5a11f6f3844392eef8bb653508a390ab5f01e43621baf.png)

-----

[^1]: Material UIの [TextField](https://material-ui.com/components/text-fields/) と同じようなイメージです。