---
title: 【Unity】VContainerとMessagePipeで再利用性を高める
date: 2022-06-10T07:11:45.662Z
draft: false
categories:
  - blog
tags:
  - Unity
  - DI
  - VContainer
  - MessagePipe
  - UniTask
---
## 再利用性の高いコード

コーディングをする上で、コードの再利用性は重要なファクターです。
でも、再利用性ってよく言うけど、ライブラリならともかく、普段の製品のコードを別の製品に再利用する機会なんてありませんよね。
それなのに、なぜ再利用性が大事なのでしょうか。

実は、コードの再利用はユニットテストで必ず行なわれます。
既存のコードをテストコードでも使用するので、再利用です。
つまり、どんなコードでもテストのために1度は再利用されるのです。
Unityでのゲーム開発も当然テストは必要ですので、再利用性を重視しなくてはなりません。

## DIの必要性

ユニットテストでは1つの機能ごとにテストを行なうので、関係のない機能をモックやドライバで置換することをよくやります。
テスト用のコードの中にテスト対象のコードだけを抽出して配置するのです。
しかし、オブジェクト指向言語でコードを書くと、振る舞いが密結合しがちです。
密結合した状態では、テスト対象をうまく抽出することが難しくなります。

そこで、各振る舞いを分離するために、DIパターンを適用してインスタンスの入れ替えを容易にできるようにすると便利です。

## VContainer

[VContainer](https://vcontainer.hadashikick.jp/ja/)はUnity用のDIコンテナです。
有名な[Zenject](https://github.com/modesttree/Zenject)と比較するとVContainerはシンプルで軽量です。

VContainerはGameObjectをコンテナに登録することができます。
これにより、PrefabやHierarchy上のGameObjectを呼び出すことができます。
また、`MonoBehaviour`でない`EntryPoint`というオブジェクトに`Start`や`Update`を記述し、コンテナに登録することができます。

## MessagePipe

[MessagePipe](https://github.com/Cysharp/MessagePipe)はPub/Subパターンのイベント発行ができるライブラリです。
イベントの発行者と購読者の間に直接的な依存関係を持たせなくすることができます。

VContainerとの連携をするために、MessagePipe.VContainerも導入します。

## UniTask

[UniTask](https://github.com/Cysharp/UniTask)はUnityで非同期処理をするためのライブラリです。
MessagePipeでは非同期のイベント発行も可能なので、こちらも導入します。

## VContainerの使い方

`VContainer.Unity.LifetimeScope`というクラスを継承し、このクラスでコンテナの設定を行ないます。

```csharp
using UnityEngine;
using VContainer;
using VContainer.Unity;

namespace akanevrc.LibraryTest
{
    public class MainLifetimeScope : LifetimeScope
    {
        protected override void Configure(IContainerBuilder builder)
        {
            // TODO
        }
    }
}
```

`Configure`メソッドの`builder`を使用してコンテナの設定をします。

```csharp
builder.RegisterComponentInHierarchy<IBehaviourA>(); // Hierarchy上のオブジェクト
builder.RegisterComponent<IBehaviourB>(prefabB);     // Prefab
builder.Register<ITypeC>(Lifetime.Scoped);           // Plain Object（スコープ付き）
builder.RegisterEntryPoint<IEntryPointD>();          // EntryPoint
```

`RegisterXX`メソッドで各種オブジェクトを登録できます。

4つ目の`RegisterEntryPoint`は、VContainer独自のイベントループを持つEntryPointを登録できます。
最初に`Start`を実行するGameObjectを配置していなくても、このオブジェクトに初期化処理をさせることができます。

登録したオブジェクトは、コンストラクタの引数としてインジェクトするのがやりやすいですが、`MonoBehaviour`はコンストラクタを自作できないので、プロパティインジェクションをするために、プロパティに`[Inject]`属性（`VContainer.InjectAttribute`）を付加します。

## MessagePipeの使い方

登録にはVContainerの拡張メソッドを使います。

```csharp
var messagePipeOptions = builder.RegisterMessagePipe();
builder.RegisterMessageBroker<XXEvent>(messagePipeOptions);
```

実際の処理では、`IPublisher`と`ISubscriber`を使用して発行/購読を行ないます。

```csharp
public class XXBehaviour : MonoBehaviour
{
    [Inject] private IPublisher<XXEvent> publisher;

    public void OnPointerDown()
    {
        this.publisher.Publish(new XXEvent());
    }
}
```

```csharp
public class XXEventHandler : IDisposable
{
    private ISubscriber<XXEvent> subscriber;

    private DisposableBagBuilder disposables = DisposableBag.CreateBuilder();
    private bool disposedValue = false;

    public XXHandler(ISubscriber<XXEvent> subscriber)
    {
        this.subscriber = subscriber;

        this.subscriber.Subscribe(ev => OnPointerDown()).AddTo(disposables);
    }

    public void OnPointerDown()
    {
        // イベント本体
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!disposedValue)
        {
            if (disposing)
            {
                this.disposables.Build().Dispose();
            }
            disposedValue = true;
        }
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
}
```

イベントの発行者と購読者に依存関係が無いことが分かります。

## まとめ

VContainerとMessagePipeで疎結合な再利用性の高いコードを書けます。
整理されたコードはバグの減少にも繋がるので、積極的に利用していきましょう。
