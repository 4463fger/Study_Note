[toc]

# Addressables使用指南

> 这里对于怎么去添加资源不做讲解，具体见其他资源解释
>
> 只对笔者在学习中所了解的API作笔记
>
> [官方文档](https://docs.unity.cn/Packages/com.unity.addressables@1.16/manual/LoadingAddressableAssets.html)

## 一：介绍

### 什么是Addressables

> Unity官方提供的资源管理系统，用于高效管理、加载和卸载资源

<span style="color: Orange;"> **优势**</span>：

- 依赖自动管理：自动处理资源间的依赖关系（如预制体依赖材质、动画）
- 灵活加载：支持本地、远程（CDN）、分块加载
- 内存优化：通过引用计数自动卸载未被使用的资源，减少内存占用
- 热更新支持：可发布增量更新，无需重装应用

## 二：API

### [1]: 核心

<span style="color: Pink;"> **AsyncOperationHandle**</span> ：用于跟踪资源加载、卸载和其他异步任务的进度和结果

#### 概念

- 在每次调用`Addressables.LoadAssetAsync`、`Addressables.InstantiateAsync`或其他异步方法的时候，就会返回一个` `。
- 而通过`AsyncOperationHandle`，能够获取操作的进度、结果、状态、并监听操作完成事件

#### Main

**基本属性**

- `Status`： 操作状态，类型为``AsyncOperationStatus` 。
- `Result` ： 操作成功后的结果对象(如加载的资源示例)
- `OperationException`： 操作失败时的异常信息
- `Task`： 返回一个`Task<T>`对象、用于与`async/await`结合使用

**事件**

- `Completed` ： 操作完成后触发的事件 ， 

**方法**

- `WaitForCompletion`：阻塞主线程直到操作完成，用来做同步操作
- `Release`：释放资源引用。``Addressables` 使用引用计数机制，多次加载同一资源需多次释放。

### [2]: 加载资源(基于官方实例)

1. `LoadAssetAsync<T>` ：异步加载单个资源
   
   - 这里用携程去处理资源
   
     ```csharp
     IEnumerator LoadGameObjectAndMaterial()
     {
         // 加载游戏物体
         AsyncOperationHandle<GameObject> goHandle = Addressables.LoadAssetAsync<GameObject>("gameObjectKey");
         yield return goHandle;
         if(goHandle.Status == AsyncOperationStatus.Succeeded)
         {
             GameObject obj = goHandle.Result;
             //etc...
         }
     
         // 加载材质
         var locationHandle =Addressables.LoadResourceLocationsAsync("materialKey");
         yield return locationHandle;
         AsyncOperationHandle<Material> matHandle = Addressables.LoadAssetAsync<Material>(locationHandle.Result[0]);
         yield return matHandle;
         if (matHandle.Status == AsyncOperationStatus.Succeeded)
         {
             Material mat = matHandle.Result;
             //etc...
         }
     
         // 不使用就释放
         Addressables.Release(goHandle);
         Addressables.Release(matHandle);
     }
     ```
   
     
   
2. `LoadAssetsAsync`



## 三：注意事项

> 确保资源在不再使用时调用 `Addressables.Release`
> 避免内存泄漏：不要长时间持有资源引用，<span style="color: Red;">**及时释放 ！！！**</span>

## 四：基于Addressables的资源加载系统

```csharp
/*
 * ┌──────────────────────────────────┐
 * │  描    述: 资源加载系统_基于AddressableAssets
 * │  类    名: ResSystem.cs
 * │  创    建: By 4463fger
 * └──────────────────────────────────┘
 */

using System;
using UnityEngine;
using System.Collections.Generic;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

namespace Study
{
    public static class ResSystem
    {
        #region 游戏物体

        /// <summary>
        /// 加载Unity资源  如AudioClip Sprite 预制体
        /// 要注意，资源不在使用时候，需要调用一次Release
        /// </summary>
        /// <param name="assetName">AB资源名称</param>
        public static T LoadAsset<T>(string assetName) where T : UnityEngine.Object
        {
            return Addressables.LoadAssetAsync<T>(assetName).WaitForCompletion();
        }

        /// <summary>
        /// 异步加载Unity资源 AudioClip Sprite GameObject(预制体)
        /// </summary>
        /// <typeparam name="T">资源类型</typeparam>
        /// <param name="assetName">AB资源名称</param>
        /// <param name="callBack">回调函数</param>
        public static void LoadAssetAsync<T>(string assetName ,Action<T> callBack)
        {
            Addressables.LoadAssetAsync<T>(assetName).Completed += (handle) =>
            {
                OnLoadAssetAsyncCompleted(handle,callBack);
            };
        }

        private static void OnLoadAssetAsyncCompleted<T>(AsyncOperationHandle<T> handle, Action<T> callBack)
        {
            callBack?.Invoke(handle.Result);
        }

        /// <summary>
        /// 同步加载指定Key的所有资源
        /// 注意:批量加载时，如果释放资源要释放掉handle，直接去释放资源是无效的
        /// </summary>
        /// <typeparam name="T">加载类型</typeparam>
        /// <param name="keyName">一般是Lable</param>
        /// <param name="handle">用来Release时使用</param>
        /// <param name="callBackOnEveryOne">注意这里是针对每一个资源的回调</param>
        /// <returns>所有资源</returns>
        public static IList<T> LoadAssets<T>(string keyName, out AsyncOperationHandle<IList<T>> handle,
            Action<T> callBackOnEveryOne = null) where T : UnityEngine.Object
        {
            handle = Addressables.LoadAssetsAsync<T>(keyName,callBackOnEveryOne,true);
            return handle.WaitForCompletion();
        }

        /// <summary>
        /// 异步加载指定Key的所有资源
        /// 注意1:批量加载时，如果释放资源要释放掉handle，直接去释放资源是无效的
        /// 注意2:回调后使用callBack中的参数使用(.Result)即可访问资源列表
        /// </summary>
        /// <typeparam name="T">加载类型</typeparam>
        /// <param name="keyName">一般是lable</param>
        /// <param name="callBack">所有资源列表的统一回调，注意这是很必要的，因为Release时需要这个handle</param>
        /// <param name="callBackOnEveryOne">注意这里是针对每一个资源的回调,可以是Null</param>
        public static void LoadAssetAsync<T>(string keyName, Action<AsyncOperationHandle<IList<T>>> callBack,
            Action<T> callBackOnEveryOne = null) where T : UnityEngine.Object
        {
            Addressables.LoadAssetsAsync<T>(keyName, callBackOnEveryOne).Completed += callBack;
        }
        
        /// <summary>
        /// 释放资源
        /// </summary>
        /// <typeparam name="T">对象类型</typeparam>
        /// <param name="obj">具体对象</param>
        public static void UnLoadAsset<T>(T obj)
        {
            Addressables.Release(obj);
        }

        /// <summary>
        /// 卸载因为批量加载而产生的handle
        /// </summary>
        /// <typeparam name="TObject"></typeparam>
        /// <param name="handle"></param>
        public static void UnLoadAsset<TObject>(AsyncOperationHandle<TObject> handle)
        {
            Addressables.Release(handle);
        }

        public static bool UnLoadInStance(GameObject obj)
        {
            return Addressables.ReleaseInstance(obj);
        }

        #endregion
    }
}
```

