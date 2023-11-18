---
title: "Codeigniter におけるバリデーションルールの設定ミスを動作確認前に検知するためのアプローチ"
date: "2023-11-18T13:39:00+09:00"
description: CodeIgniter におけるバリデーションルールの設定ミスを動作確認前に検知する方法を紹介しています。
   バリデーションルールの連想配列を生成するためのクラスを用意しつつ、エディタの自動補完を利用します。
---

{{< alert type="success" >}}
今回紹介しているコードはこちらにも記載しています。  
https://github.com/noritakaIzumi/codeigniter-validation-extended/tree/20231118/Validator
{{</ alert >}}

CodeIgniter のバリデーションルールは通常、

```php
$validation->setRule('username', 'Username', 'required|max_length[30]|min_length[3]');
```

のように文字列で指定されますが、

```php
$validation->setRule('username', 'Username', 'requaired|max_langth[3O]|mln_length[3]');
                                                  ^          ^      ^   ^
```

のように設定を間違えていた場合、実際に動作させるまでミスに気づくことができません。
（テストコードを書いていれば別ですが、ここでは書いていないものとします）

そこで、バリデーションルールを生成するためのクラスを用意し、ルール名やパラメータを間違えた場合にエディタの静的解析で気付くための方法を考えます。  
また、エディタの自動補完もうまく使えるようにします。

## 環境

- CodeIgniter 4.4.3
- PHP 8.2.12

## 実装

### 目指す形

クラスを使って生成したルールを変換したときに、
[Setting Custom Error Messages](https://www.codeigniter.com/user_guide/libraries/validation.html#setting-custom-error-messages)
のようにフィールドラベルの付いた連想配列が生成されるのを目標とします。  
`rules` はパイプ区切りの文字列もしくは配列で指定できますが、今回は配列で生成することとします。

```php
$rules = [
    'username' => [
        'label'  => 'Username',
        'rules'  => ['required', 'max_length[30]', 'min_length[3]'],
        'errors' => [
            'required' => 'All accounts must have {field} provided',
        ],
    ],
    'password' => [
        'label'  => 'Password',
        'rules'  => ['required', 'max_length[255]', 'min_length[8]', 'alpha_numeric_punct'],
        'errors' => [
            'min_length' => 'Your {field} is too short. You want to get hacked?',
        ],
    ],
];
```

### 開発イメージ

各フィールドは `Field` クラスを用意し、コンストラクタで `label`, `rules`, `errors` を引数に取るのが良いかもしれません。  
しかし、`errors` の各キーは `rules` で指定されるものに依存するので、エラーメッセージ込みのルール設定を渡せるようにします。

`rules` 向けに `FieldRules` クラスを用意し、インスタンスを生成した後で各ルールをメソッドチェーンで追加できるようにします。

`Field` クラスに `export` メソッドを用意し、目指している連想配列を出力できるようにします。

すなわち、以下のような書き方で実装できるものとします。

```php
# username
$usernameFieldRules = new FieldRules();
$usernameFieldRules
    ->required(message: 'All accounts must have {field} provided')
    ->maxLength(30)
    ->minLength(3);
$usernameField = new Field(name: 'username', label: 'Username', rules: $usernameFieldRules);

# password
$passwordFieldRules = new FieldRules();
$passwordFieldRules
    ->required()
    ->minLength(8, message: 'Your {field} is too short. You want to get hacked?')
    ->maxLength(255)
    ->alphaNumericPunct();
$passwordField = new Field(name: 'password', label: 'Password', rules: $passwordFieldRules);

$rulesCreator = new RulesCreator();
$rulesCreator
    ->addField($usernameField)
    ->addField($passwordField);

$validation = \Config\Services::validation();
$validation->setRules($rulesCreator->export());
```

### `FieldRules` クラス

`FieldRules` クラスには各ルール名に対応したメソッドを用意します。  
ここでは例として `minLength` メソッドを実装します。

```php
class FieldRules
{
    protected array $rules = [];
    protected array $errors = [];

    public function getRules(): array
    {
        return $this->rules;
    }

    public function getErrors(): array
    {
        return $this->errors;
    }

    // min_length ルールはしきい値となる文字数が必要なので、int 型の引数を用意します。
    public function minLength(int $length, string $message = ''): static
    {
        $this->rules[] = "min_length[$length]";
        if ($message !== '') {
            $this->errors['min_length'] = $message;
        }
        // メソッドチェーンに対応させるため、$this を返します。
        return $this;
    }
}
```

他のメソッドも同様に実装します。

### `Field` クラス

`Field` クラスは
`name`, `label` のほかに先ほど作成した `FieldRules` クラスのインスタンスを持たせた value object とします。

```php
readonly class Field
{
    /**
     * @param string $name
     * @param string $label
     * @param FieldRules $rules
     */
    public function __construct(public string $name, public string $label, public FieldRules $rules)
    {
    }
}
```

### `RulesCreator` クラス

`RulesCreator` クラスには `addField` メソッドを用意して各フィールドの情報を持てるようにします。  
また、CodeIgniter のバリデーターに渡せる形に変換する `export()` メソッドも用意します。

```php
class RulesCreator
{
    /**
     * @var Field[]
     */
    protected array $fields = [];

    public function addField(Field $field): static
    {
        $this->fields[] = $field;
        return $this;
    }

    public function export(): array
    {
        $rules = [];
        foreach ($this->fields as $field) {
            $rule = [
                'label' => $field->label,
                'rules' => $field->rules->getRules(),
            ];
            $errors = $field->rules->getErrors();
            if ($errors !== []) {
                $rule['errors'] = $errors;
            }

            $rules[$field->name] = $rule;
        }

        return $rules;
    }
}
```

### 実行

以下のようにして実行することができます。

```php
$validation = \Config\Services::validation();
$validation->setRules($rulesCreator->export());
if (! $validation->run($data)) {
    // handle validation errors
}
```

## まとめ

今回のようにフレームワークをカスタマイズすることは、開発スピードの改善にもつながりますし、エディタの機能と組み合わせて実装ミスなどを減らすことにもつながります。
その一方で、フレームワークを熟知していない人にとっては、どれがカスタマイズした機能でどれがフレームワーク純正の機能なのか、分かりにくくなることもあります。
開発の心得やマニュアル・環境などを整備し、多様なエンジニアが効率的な開発をできるように工夫することが必要だと思います。
