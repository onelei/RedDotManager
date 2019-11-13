# Unity红点系统的实现

在使用Unity开发游戏的时候经常用到红点系统，当玩家点击之后，或者收到服务器数据之后，都需要刷新红点的显示。如果每个人都自己写自己的红点模块，会增加不少的重复任务量，因此迫切需要一个通用的红点系统，其他模块只需要编写自己模块的红点类型和对应的是否显示红点的判断即可。因此RedDotManager应运而生。

## 案例

下面通过一个邮件红点来演示如何操作使用。如下图，当点击系统邮件按钮和玩家邮件按钮之后，对应按钮上面的红点会消除，两个按钮分别点击之后，所有邮件按钮上面的红点才消除。

![Video_2019-11-12_205423](https://github.com/onelei/RedDotManager/blob/master/Images/Video_2019-11-12_205423.gif)

我们在每个按钮下面都拖上一个带有RedDotItem脚本的红点Prefab。

![图片2](https://github.com/onelei/RedDotManager/blob/master/Images/图片2.png)

上图中的“Image”就是红点，我们只要在逻辑层控制该“Image”的显示即可。

图中可以看到该红点可以设置多个红点类型，也就是支持一个红点由多个含义。比如所有邮件按钮上的红点就是由系统邮件和玩家邮件一起控制显示，当任意一个返回true，显示红点的时候，该红点就需要一直显示。直到该RedDotTypes下面的所有逻辑都返回false的时候，红点才不显示。

### 红点类型--RedDotType

C#

```
/**
 * 红点类型的枚举;
 * RedDotType.cs
 * 
 * Created by Onelei 12/11/2017 10:28:04 AM. All rights reserved.
**/

namespace RedDot
{
    public enum RedDotType
    {
        None = 0,
        Email_UnReadSystem = 1,
        Email_UnReadPlayer = 2,
    }
}
```

### 红点组件--RedDotItem

我们看下红点身上的组件RedDotItem.cs

```
/**
 * 红点物体;
 * RedDotItem.cs
 * 
 * Created by Onelei 12/11/2017 10:21:47 AM. All rights reserved.
**/
using UnityEngine;
using UnityEngine.UI;
using System.Collections.Generic;

namespace RedDot
{
    public class RedDotItem : MonoBehaviour
    {  
        /// <summary>
        /// 红点物体;
        /// </summary>
        public GameObject Go_RedDot;
        /// <summary>
        /// 红点类型;
        /// </summary>
        public List<RedDotType> redDotTypes;

        private bool bCachedRedDot = false;

        void Start()
        {
            //注册红点;
            RedDotManager.Instance.RegisterRedDot(redDotTypes, this);
            //设置红点;
            bool bRedDot = RedDotManager.Instance.Check(redDotTypes);
            SetData(bRedDot, true);
        }

        void OnDestroy()
        {
            //取消注册红点;
            RedDotManager.Instance.UnRegisterRedDot(this);
        }

        /// <summary>
        /// 设置红点显示;
        /// </summary>
        /// <param name="bRedDot"></param>
        public void SetData(bool bRedDot, bool bForceRefresh = false)
        {
            if (bForceRefresh)
            {
                Go_RedDot.SetActive(bRedDot);
                bCachedRedDot = bRedDot;
                return;
            }

            if (bCachedRedDot != bRedDot)
            {
                Go_RedDot.SetActive(bRedDot);
                bCachedRedDot = bRedDot;
            }
        }
        /// <summary>
        /// 获取当前物体挂载的所有红点;
        /// </summary>
        /// <returns></returns>
        public List<RedDotType> GetRedDotTypes()
        {
            return this.redDotTypes;
        }

    }
}
```

在Start和OnDestroy的时候，注册和反注册红点消息的监听。在Start的时候根据自身的红点类型来判断是否显示红点即可。此时红点的GameObject也注册进去了，所以，后面只要根据红点是否显示，直接在管理类里面控制刷新显示即可。

RedDotManager.cs是红点的管理类，该类主要用来将注册进来的红点使用Dictionary统一管理。

### 红点初始化--Initilize

```
        /// <summary>
        /// 初始化红点系统(注意只需要初始化一次);
        /// </summary>
        public void Initilize()
        {
           RedDotConditionDict.Clear();

            // 添加红点数据判断;
            RedDotConditionDict.Add(RedDotType.Email_UnReadSystem, new RedDot_EmailUnReadSystem());
            RedDotConditionDict.Add(RedDotType.Email_UnReadPlayer, new RedDot_EmailUnReadPlayer());

        }
```

我们需要一个通用的红点判断的基类，因此设计RedDotBase如下

```
/**
 * 红点数据类-基类;
 * RedDotBase.cs
 * 
 * Created by Onelei 12/11/2017 10:23:47 AM. All rights reserved.
**/

namespace RedDot
{
    public abstract class RedDotBase
    {
        /// <summary>
        /// 是否显示红点(true表示显示,false表示不显示;)
        /// </summary>
        /// <param name="?"></param>
        /// <returns></returns>
        public virtual bool ShowRedDot(object[] objs)
        {
            return false;
        }
    }
}
```

不同的红点判断继承该父类，子类重写ShowRedDot函数即可。我们看下红点判断函数的实现

```
/**
 * 未读邮件红点判断逻辑-玩家红点;
 * RedDot_EmailUnReadPlayer.cs
 * 
 * Created by Onelei 12/11/2017 10:25:00 AM. All rights reserved.
**/
using RedDot;

public class RedDot_EmailUnReadPlayer : RedDotBase
{
    public override bool ShowRedDot(object[] objs)
    {
        return MailManager.Instance.IsPlayerRedDot();
    }
}
```

```
/**
 * 未读邮件红点判断逻辑-系统红点;
 * RedDot_EmailUnRead.cs
 * 
 * Created by Onelei 12/11/2017 10:25:00 AM. All rights reserved.
**/
using RedDot;

public class RedDot_EmailUnReadSystem : RedDotBase
{
    public override bool ShowRedDot(object[] objs)
    {
        return MailManager.Instance.IsSystemRedDot();
    }
}
```

我们通过MailManager里面的数据类判断是否显示红点，ShowRedDot函数的参数是object数组，方便检查红点的时候传入参数。比如判断某个英雄是否可以升级，就可以传入英雄的ID。

### 红点注册--RegisterRedDot

```
        /// <summary>
        /// 添加红点界面;
        /// </summary>
        /// <param name="redDotType"></param>
        /// <param name="item"></param>
        private void RegisterRedDot(RedDotType redDotType, RedDotItem item)
        {
            if (RedDotObjDict.ContainsKey(redDotType))
            {
                RedDotObjDict[redDotType].Add(item);
            }
            else
            {
                List<RedDotItem> items = new List<RedDotItem>();
                items.Add(item);
                RedDotObjDict.Add(redDotType, items);
            }
        }
```

### 红点检查--Check是否显示

```
        /// <summary>
        /// 检查红点提示,内部调用;
        /// </summary>
        /// <param name="redDotType"></param>
        /// <returns></returns>
        private bool Check(RedDotType redDotType, object[] objs)
        {
            if (RedDotConditionDict.ContainsKey(redDotType))
            {
                return RedDotConditionDict[redDotType].ShowRedDot(objs);
            }
            return false;
        }
```

### 红点系统管理类--RedDotManager

完整代码如下：

```
/**
 * 红点系统管理类;
 * RedDotManager.cs
 * 
 * Created by Onelei 12/11/2017 10:21:47 AM. All rights reserved.
**/

using System.Collections.Generic;

namespace RedDot
{
    public class RedDotManager
    {
        private static RedDotManager _instance;
        public static RedDotManager Instance
        {
            get
            {
                if (null == _instance)
                {
                    _instance = new RedDotManager();
                }
                return _instance;
            }
        }

        /// <summary>
        /// 红点数据;
        /// </summary>
        Dictionary<RedDotType, RedDotBase> RedDotConditionDict = new Dictionary<RedDotType, RedDotBase>();
        /// <summary>
        /// 红点物体;
        /// </summary>
        Dictionary<RedDotType, List<RedDotItem>> RedDotObjDict = new Dictionary<RedDotType, List<RedDotItem>>();
        /// <summary>
        /// 初始化红点系统(注意只需要初始化一次);
        /// </summary>
        public void Initilize()
        {
           RedDotConditionDict.Clear();

            // 添加红点数据判断;
            RedDotConditionDict.Add(RedDotType.Email_UnReadSystem, new RedDot_EmailUnReadSystem());
            RedDotConditionDict.Add(RedDotType.Email_UnReadPlayer, new RedDot_EmailUnReadPlayer());

        }
         
        /// <summary>
        /// 注册红点;
        /// </summary>
        /// <param name="redDotType"></param>
        /// <param name="item"></param>
        public void RegisterRedDot(List<RedDotType> redDotTypes, RedDotItem item)
        {
            for (int i = 0; i < redDotTypes.Count; i++)
            {
                RegisterRedDot(redDotTypes[i], item);
            }
        }
        /// <summary>
        /// 取消注册红点;
        /// </summary>
        /// <param name="item"></param>
        public void UnRegisterRedDot(RedDotItem item)
        {
            Dictionary<RedDotType, List<RedDotItem>>.Enumerator itor = RedDotObjDict.GetEnumerator();
            while (itor.MoveNext())
            {
                List<RedDotItem> redDotItems = itor.Current.Value;
                if (redDotItems.Contains(item))
                {
                    redDotItems.Remove(item);
                    break;
                }
            }
        }

        /// <summary>
        /// 检查红点提示;
        /// </summary>
        /// <param name="redDotType"></param>
        /// <returns></returns>
        public bool Check(List<RedDotType> redDotTypes, object[] objs = null)
        {
            for (int i = 0; i < redDotTypes.Count; i++)
            {
                //只要有一个需要点亮,就显示;
                if (Check(redDotTypes[i], objs))
                {
                    return true;
                }
            }
            return false;
        }

        /// <summary>
        /// 更新该类型的所有红点组件;
        /// </summary>
        /// <param name="redDotType"></param>
        /// <returns></returns>
        public void NotifyAll(RedDotType redDotType, object[] objs = null)
        {
            if (RedDotObjDict.ContainsKey(redDotType))
            {
                for (int i = 0; i < RedDotObjDict[redDotType].Count; i++)
                {
                    RedDotItem item = RedDotObjDict[redDotType][i];
                    if (item != null)
                    {
                        List<RedDotType> redDotTypes = item.GetRedDotTypes();
                        bool bCheck = Check(redDotTypes, objs);
                        item.SetData(bCheck);
                    }
                }
            }
        }
        #region private
        /// <summary>
        /// 添加红点界面;
        /// </summary>
        /// <param name="redDotType"></param>
        /// <param name="item"></param>
        private void RegisterRedDot(RedDotType redDotType, RedDotItem item)
        {
            if (RedDotObjDict.ContainsKey(redDotType))
            {
                RedDotObjDict[redDotType].Add(item);
            }
            else
            {
                List<RedDotItem> items = new List<RedDotItem>();
                items.Add(item);
                RedDotObjDict.Add(redDotType, items);
            }
        }
        /// <summary>
        /// 检查红点提示,内部调用;
        /// </summary>
        /// <param name="redDotType"></param>
        /// <returns></returns>
        private bool Check(RedDotType redDotType, object[] objs)
        {
            if (RedDotConditionDict.ContainsKey(redDotType))
            {
                return RedDotConditionDict[redDotType].ShowRedDot(objs);
            }
            return false;
        } 
        #endregion
    }
}
```

## 测试

编写一个测试函数，在Start函数里面添加按钮的回调

```
    // Use this for initialization
    void Start()
    {
        // 设置按钮回调;
        Button_EmailSystem.onClick.AddListener(OnClickEmailSystem);
        Button_EmailPlayer.onClick.AddListener(OnClickEmailPlayer);

    }
```

我们需要在按钮点击的时候，计算该红点类型所对应的的红点逻辑，同时刷新所有绑定了该红点类型的RedDotItem身上的红点。

```
    /// <summary>
    /// 系统按钮的回调;
    /// </summary>
    void OnClickEmailSystem()
    {
        // 点击之后表示邮件读取过了,设置邮件为已读状态;
        // (注意这里不能够设置红点的显隐,因为是有数据控制的,所以要控制数据那边的红点逻辑);
        // RedDot.RedDotManager.Instance.SetData(RedDot.RedDotType.Email_UnRead,false);

        RedDot.MailManager.Instance.SetSystemRedDot(false);
        RedDot.RedDotManager.Instance.NotifyAll(RedDot.RedDotType.Email_UnReadSystem);
    }

    /// <summary>
    /// 玩家按钮的回调;
    /// </summary>
    void OnClickEmailPlayer()
    {
        // 点击之后表示邮件读取过了,设置邮件为已读状态;
        // (注意这里不能够设置红点的显隐,因为是有数据控制的,所以要控制数据那边的红点逻辑);
        // RedDot.RedDotManager.Instance.SetData(RedDot.RedDotType.Email_UnRead,false);

        RedDot.MailManager.Instance.SetPlayerRedDot(false);
        RedDot.RedDotManager.Instance.NotifyAll(RedDot.RedDotType.Email_UnReadPlayer);
    }
```

## 总结

我们将红点的数据和表现分离，通过RedDotItem来控制红点的显示。RedDotItem是一个单独的Prefab，我们可以设计不同的红点，比如红点上面是否有数字等，都可以在一个RedDotItem上面进行扩展。同时还支持一个红点类型绑定在不同的Prefab上面，一个Prefab上面绑定多个红点。这样一对多的关系，我们使用Dictionary来方便管理，Key就是红点类型，Value是一个List，里面包含了该类型绑定的所有RedDotItem。只要取出来对应的红点类型的红点是否和之前不一样，不一样就刷新该Key对应的Value的所有RedDotItem红点的显示即可。