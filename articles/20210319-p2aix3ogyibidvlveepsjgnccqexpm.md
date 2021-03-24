---
title: "aws-sdk-php + PartiQL"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [dynamodb, partiql, aws-sdk-php]
published: true
---

# What's

[DynamoDB-local + PartiQLが可能になった](https://aws.amazon.com/jp/about-aws/whats-new/2021/02/you-now-can-use-partiql-with-dynamodb-local-to-query-insert-update-and-delete-table-data-in-amazon-dynamodb/)ので試しているところ。
コードは[Slim4-Skelton](https://github.com/odan/slim4-skeleton)をDynamoDB用に書き換えたもの。

↓リポジトリはこちら
https://github.com/watarukura/php_vscode_project_template

## テーブル定義

usersテーブル。

```yaml
$ aws --endpoint http://localhost:8000 dynamodb delete-table --table-name users
TableDescription:
  AttributeDefinitions:
  - AttributeName: id
    AttributeType: N
  BillingModeSummary:
    BillingMode: PAY_PER_REQUEST
    LastUpdateToPayPerRequestDateTime: '2021-02-17T13:02:33.967000+09:00'
  CreationDateTime: '2021-02-17T13:02:33.967000+09:00'
  ItemCount: 1
  KeySchema:
  - AttributeName: id
    KeyType: HASH
  ProvisionedThroughput:
    LastDecreaseDateTime: '1970-01-01T09:00:00+09:00'
    LastIncreaseDateTime: '1970-01-01T09:00:00+09:00'
    NumberOfDecreasesToday: 0
    ReadCapacityUnits: 0
    WriteCapacityUnits: 0
  TableArn: arn:aws:dynamodb:ddblocal:000000000000:table/users
  TableName: users
  TableSizeBytes: 66
  TableStatus: ACTIVE
```

counterテーブル。

```yaml
$ aws --endpoint http://localhost:8000 dynamodb delete-table --table-name counter
TableDescription:
  AttributeDefinitions:
  - AttributeName: id
    AttributeType: S
  BillingModeSummary:
    BillingMode: PAY_PER_REQUEST
    LastUpdateToPayPerRequestDateTime: '2021-02-17T13:02:33.947000+09:00'
  CreationDateTime: '2021-02-17T13:02:33.947000+09:00'
  ItemCount: 1
  KeySchema:
  - AttributeName: id
    KeyType: HASH
  ProvisionedThroughput:
    LastDecreaseDateTime: '1970-01-01T09:00:00+09:00'
    LastIncreaseDateTime: '1970-01-01T09:00:00+09:00'
    NumberOfDecreasesToday: 0
    ReadCapacityUnits: 0
    WriteCapacityUnits: 0
  TableArn: arn:aws:dynamodb:ddblocal:000000000000:table/counter
  TableName: counter
  TableSizeBytes: 16
  TableStatus: ACTIVE
```

## get-item

before

```php
    /**
     * Get user by the given user id.
     *
     * @param int $userId The user id
     *
     * @return UserReaderData The user data
     *
     * @throws DomainException
     * @throws Exception
     */
    public function getUserById(int $userId): UserReaderData
    {
        try {
            $marshaler = new Marshaler();
            $key = $marshaler->marshalJson((string)json_encode([
                'id' => $userId
            ]));
            $row = $this->client->getItem([
                'TableName'      => 'users',
                'Key'            => $key,
            ]);
            if (!$row->get('Item')) {
                throw new DomainException(sprintf('User not found: %s', $userId));
            }
        } catch (Exception $exception) {
            throw $exception;
        }
        // Map array to data object
        $user = new UserReaderData();
        $user->id = (int)$row['Item']['id']['N'];
        $user->username = (string)$row['Item']['username']['S'];
        $user->first_name = (string)$row['Item']['first_name']['S'];
        $user->last_name = (string)$row['Item']['last_name']['S'];
        $user->email = (string)$row['Item']['email']['S'];

        return $user;
```

after

```php
    /**
     * Get user by the given user id.
     *
     * @param int $userId The user id
     *
     * @return UserReaderData The user data
     *
     * @throws DomainException
     * @throws Exception
     */
    public function getUserById(int $userId): UserReaderData
    {
        try {
            $result = $this->client->executeStatement([
                'Parameters' => [
                    [
                        'N' => $userId
                    ]
                ],
                'Statement'  => 'SELECT * FROM users WHERE id = ?'
            ]);
            $items = $result->get('Items') ?? [];
            if (!$items) {
                throw new DomainException(sprintf('User not found: %s', $userId));
            } else {
                $marshaler = new Marshaler();
                $result_item = json_decode(
                    $marshaler->unmarshalJson($items[0]),
                    true
                );
            }
        } catch (Exception $exception) {
            throw $exception;
        }

        // Map array to data object
        $user = new UserReaderData();
        $user->id = $result_item['id'];
        $user->username = $result_item['user_name'];
        $user->first_name = $result_item['first_name'];
        $user->last_name = $result_item['last_name'];
        $user->email = $result_item['email']

        return $user;
    }
```

## put-item

before

```php
    /**
     * Insert user row.
     *
     * @param UserCreatorData $user The user
     *
     * @return int The new ID
     * @throws Exception
     */
    public function insertUser(UserCreatorData $user): int
    {
        $id = $this->incrementUserId();

        try {

            $marshaler = new Marshaler();
            $item = $marshaler->marshalJson((string)json_encode([
                'id'         => $id,
                'username'   => $user->username,
                'first_name' => $user->first_name,
                'last_name'  => $user->last_name,
                'email'      => $user->email,
            ]));

            $this->client->putItem([
                'TableName' => 'users',
                'Item'      => $item,
            ]);
        } catch (Exception $exception) {
            throw $exception;
        }

        return $id;
    }
```

after

```php
    /**
     * Insert user row.
     *
     * @param UserCreatorData $user The user
     *
     * @return int The new ID
     * @throws Exception
     */
    public function insertUser(UserCreatorData $user): int
    {
        $id = $this->incrementUserId();

        try {
            $this->client->executeStatement([
                'Parameters' => [
                    [
                        'N' => $id,
                    ],
                    [
                        'S' => $user->username,
                    ],
                    [
                        'S' => $user->first_name,
                    ],
                    [
                        'S' => $user->last_name,
                    ],
                    [
                        'S' => $user->email
                    ]
                ],
                'Statement'  => "INSERT INTO users
                                 VALUE {
                                    'id': ?,
                                    'user_name': ?,
                                    'first_name': ?,
                                    'last_name': ?,
                                    'email': ?
                                 }"
            ]);
        } catch (Exception $exception) {
            throw $exception;
        }

        return $id;
    }
```

## update-item

updateは元ネタがないのでスクラッチ。

```php
    /**
     * Update user row.
     *
     * @param UserUpdaterData $userUpdaterData
     *
     * @return UserUpdaterData The user data
     *
     * @throws Exception
     */
    public function updateUser(UserUpdaterData $userUpdaterData): UserUpdaterData
    {
        $statement = 'UPDATE users ';
        $parameters = [];
        foreach ($userUpdaterData->jsonSerialize() as $k => $v) {
            if ($k === 'id') {
                // TODO: Omit Partition Key
                continue;
            }
            if (is_null($v)) {
                continue;
            }
            $statement .= sprintf('SET %s=? ', $k);
            $parameters[] = $marshaler->marshalValue($v);
        }
        $statement .= ' WHERE id = ? RETURNING ALL NEW *';
        $parameters[] = [
            'N' => $userUpdaterData->id
        ];

        try {
            $result = $this->client->executeStatement([
                'Parameters' => $parameters,
                'Statement'  => $statement
            ]);
            $items = $result->get('Items') ?? [];
            if (!$items) {
                throw new DomainException(sprintf('User not found: %s', $userUpdaterData->id));
            } else {
                $marshaler = new Marshaler();
                $result_item = json_decode(
                    $marshaler->unmarshalJson($items[0]),
                    true
                );
            }
        } catch (Exception $exception) {
            throw $exception;
        }

        return new UserUpdaterData($userUpdaterData->id, $result_item);
    }
```

## atomic counter

こちらはbeforeも含めてスクラッチ。
counterテーブルへの初期登録が必要です。

```sh
aws --endpoint http://localhost:8000 ddb put counter '{id: "users", counter: 0}'
```

before

```php
    /**
     * @return int
     * @throws Exception
     */
    private function incrementUserId(): int
    {
        $marshaler = new Marshaler();
        $key = $marshaler->marshalItem([
            'id' => 'users'
        ]);
        $eav = $marshaler->marshalItem([
            ':increment' => 1
        ]);
        $ean = [
            '#c' => 'counter'
        ];
        $params = [
            'TableName'                 => 'counter',
            'Key'                       => $key,
            'UpdateExpression'          => 'ADD #c :increment',
            'ExpressionAttributeValues' => $eav,
            'ExpressionAttributeNames'  => $ean,
            'ReturnValues'              => 'UPDATED_NEW'
        try {
            $result = $this->client->updateItem($params);
            $attr = $result['Attributes'] ?? [];
            $counter = json_decode(
                $marshaler->unmarshalJson($attr),
                true
            );
            if ($counter) {
                return $counter['counter'];
            } else {
                throw new DomainException(sprintf('Counter not found: %s', 'users'));
        } catch (DynamoDbException $exception) {
            throw $exception;
        }
    }
```

after

```php
    /**
     * @return int
     * @throws Exception
     */
    private function incrementUserId(): int
    {
        $marshaler = new Marshaler();
        $statement = 'UPDATE counter '
                   . 'SET counter = counter + ? '
                   . 'WHERE id = ? RETURNING ALL NEW *;';
        $parameters = [
            [
                'N' => 1
            ],
            [
                'S' => 'users'
            ]
        ];

        try {
            $result = $this->client->executeStatement([
                'Parameters' => $parameters,
                'Statement'  => $statement
            ]);
            $items = $result->get('Items') ?? [];
            if (!$items) {
                throw new DomainException(sprintf('Counter not found: %s', 'users'));
            } else {
                $result_item = json_decode(
                    $marshaler->unmarshalJson($items[0]),
                    true
                );
                return $result_item['counter'];
            }
        } catch (Exception $exception) {
            throw $exception;
        }
    }
```

## 感想

Python/Java/Node.jsはNoSQL workbenchでコード生成してくれますが、PHPは生成してくれません。
無念。

https://qiita.com/watarukura/items/1c42b625aecb7aee12fa が元記事です。
textlintでfixしました。