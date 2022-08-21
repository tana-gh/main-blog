---
title: 【Unity】AnimatorControllerをEditorScriptから保存する
date: 2022-02-24T05:34:11.417Z
draft: false
categories:
  - blog
tags:
  - Unity
  - AnimatorController
---
`AnimatorController`をEditorスクリプトから保存する方法を書いておきたいと思います。

`AnimatorContoller`自体の保存は、ダーティにしてからセーブするだけです。

```csharp
// 編集後
EditorUtility.SetDirty(animatorController);
AssetDatabase.SaveAssets();
```

しかし、`AnimatorController`は作りが特殊らしく、この方法ではサブアセットが保存されません。
ここでサブアセットとは、以下のオブジェクトのことです。

- `AnimatorStateMachine`
- `AnimatorState`
- `AnimatorStateTransition`
- `AnimatorTransition`
- `BlendTree`

`AnimatorController`の保存時にこれらの変更を保存するには、以下のようにします。

```csharp
EditorUtility.SetDirty(subAssetObj);
if (AssetDatabase.GetAssetPath(subAssetObj).Length == 0)
{
    obj.hideFlags = HideFlags.HideInHierarchy;
    AssetDatabase.AddObjectToAsset(subAssetObj, assetPath);
}
```

従って、`AnimatorController`保存時の前処理として、サブアセットの登録処理として以下のような`Visitor`を用意すると良いと思います。

```csharp
private class AssetVisitor
{
    private readonly AnimatorController Controller;
    private readonly string Path;

    public AssetVisitor(AnimatorController controller)
    {
        Controller = controller;
        Path       = AssetDatabase.GetAssetPath(Controller);
    }

    public void Visit()
    {
        foreach (var x in Controller.layers) VisitStateMachine(x.stateMachine);
    }

    private void VisitStateMachine(AnimatorStateMachine src)
    {
        if (src == null) return;
        AddSubAsset(src);
        foreach (var x in src.stateMachines      ) VisitStateMachine   (x.stateMachine);
        foreach (var x in src.states             ) VisitState          (x.state);
        foreach (var x in src.anyStateTransitions) VisitStateTransition(x);
        foreach (var x in src.entryTransitions   ) VisitTransition     (x);
    }

    private void VisitState(AnimatorState src)
    {
        if (src == null) return;
        AddSubAsset(src);
        foreach (var x in src.transitions) VisitStateTransition(x);
        VisitMotion(src.motion);
    }

    private void VisitStateTransition(AnimatorStateTransition src)
    {
        if (src == null) return;
        AddSubAsset(src);
    }

    private void VisitTransition(AnimatorTransition src)
    {
        if (src == null) return;
        AddSubAsset(src);
    }

    private void VisitMotion(Motion src)
    {
        if (src == null) return;
        AddSubAsset(src);
        if (src is BlendTree tree) VisitBlendTree(tree);
    }

    private void VisitBlendTree(BlendTree src)
    {
        foreach (var x in src.children) VisitMotion(x.motion);
    }

    private void AddSubAsset(UnityEngine.Object obj)
    {
        EditorUtility.SetDirty(obj);
        if (AssetDatabase.GetAssetPath(obj).Length == 0)
        {
            obj.hideFlags = HideFlags.HideInHierarchy;
            AssetDatabase.AddObjectToAsset(obj, Path);
        }
    }
}
```