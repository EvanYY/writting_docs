# 有趣的WebAPI-MutationObserver

  最近开发chrome辅助插件时，有个需求要类似插件注入监听代码，实时监听目标页面的DOM监听数据及时上报，故使用了这个webAPI，挺有意思的大家分享下。

## 业务需求

  是一个有关外贸商城，用户需要通过埋点piwik捕获访客浏览及点击行为信息后，并将有完成订单的客户向CRM端发送订单详情（地址，购物车内容，订单编号，联系方式，CRM指定创建的唯一客户ID）。猜猜有几种实现方式咧~

## 解决问题方式

经过基本思考有两种方式

- 劫持网站XMLHttpRequest，将指定接口数据携带指定身份同步到指定地址
- 劫持指定节点DOM，解构DOM节点数据，携带指定身份同步到指定位置

## 实现方式

1. 基于现有业务在chrome插件中拓展出功能，将劫持的指定接口返回值通过插件管道绕过CSP同步数据。这个已经实现了，插件中我劫持了instagram接口，故在客户无感情况遍历获取聊天组、帖子列表、消息、历史消息、评论、历史评论，同步到指定位置的CRM系统，instagram也做了CSP安全策略，当时着实费了些手脚，但是有惊无险的解决了，有机会在跟大家分享下。

2. 稍微麻烦点，定制化实时监听DOM节点数据变化（地址修改，购物车内容修改，发票订单修改，游客端创建订单等）并利用localStore作为临时储存容器，初始化或结束时删除容器仅需注意当前页面重载时是否触发机制就可以了。

比较下思路优缺点

- 方案一优点是简单仅需专注业务即可，但是比较依赖chrome插件容易造成数据丢失，而且遇到比较古老的多页商城或者业务比较复杂奇葩的逻辑需要了解数据逻辑学习成本比较高、不适用类似JSP写的渲染页面，例如做外贸常用的magento 商城服务

- 方案二优点是无浏览器限制，获取业务数据比较简单，缺点是结构DOM比较麻烦，万一目标网站节点标识变了则需要拓展监听目标，客户要是修改了DOM容易造成数据混乱问题，而且最最关键是得需要网站后台同意的权限，进行CSP放行，也允许注入业务代码~~~万幸，商城们在这方面权限是放开的，仅需对后台简单的配置即可，而且关键数据可以在某个悄悄储存的地方获取，阿弥陀佛。

## 有趣的 MutationObserver

废话说了不少终于进入正题，如各位所见，最终选择了第二种，因为客户愿意开放权限，而且类似那种magento这种商城节点是固定的所以咯，就开始了我们的 <font color=#ff502c>MutationObserver</font>

1. 什么是 [MDN: MutationObserver 介绍](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)

2. 总的来说就是监听节点内容变化包括内容、子节点属性变化例如style，class~~~

## 容器及业务设计

其上游就不用详细介绍了，直接说容器和业务怎么处理话不多说直接上代码

```javascript {.class1 .class}
import * as behaviors from "./behavior"; // 行为对象根据cbName加载相应的业务逻辑
export class CheckOutAddressBehavior { // 多容器实例类
  constructor(targetDOMName, cbName, cb) {
    this.targetDOMName = targetDOMName; // 目标跟节点唯一标识
    this.cbName = cbName; // behaviors 业务键 名
    this.targetNode = null; // 监听的目标根节点，尽量找中间的唯一的标识
    this.config = { // MutationObserver 配置
      attributes: true,
      childList: true,
      subtree: true
    };
    this.cb = cb; // 指定回调可忽略
    this.observer = null;
  }
  check() {
    return 
  }
  getDom() {
    return document.querySelector(this.targetDOMName); // 写的时候不需要额外处理，如果节点变更则 targetDOMName 应为数组并做降级处理
  }
  start() { // 开始执行
    this.targetNode = this.getDom(); // 因为是在业务执行前注入又或者节点是异步创建故需要捕获
    if (!this.targetNode) {
      setTimeout(() => {
        this.start();
      }, 200);
      // console.warn(`${this.targetDOMName} is not exit`);
    } else {
      this.assignEvent();
      setTimeout(() => {
        if (this.cbName && typeof behaviors[this.cbName] === "function")
          behaviors[this.cbName](); // 初始化成功 创建所需数据结构并初始化一次业务
      }, 100);
      if (typeof this.cb === "function") {
        this.cb(); // cb 若存在则执行，我这没调用
      }
    }
  }
  assignEvent() { // 注册监听事件
    if (!window.MutationObserver)
      throw Error(
        "MutationObserver is not exit. Please check your browser version (from XHL)"
      );
    let _this = this;
    this.observer = new MutationObserver(function(mutationsList) {
      for (let mutation of mutationsList) {
        if (mutation.type === "childList" || mutation.type === "attributes") {
          if (_this.cbName && typeof behaviors[_this.cbName] === "function") {
            behaviors[_this.cbName](mutation);
          } else {
            throw Error("cbName is not exit in behaviors", behaviors);
          }
        }
      }
    });
    this.observer.observe(this.targetNode, this.config); // 注册监听 事件
  }
}
```

```javascript {.class1 .class}
// behaviors 设计
import { debounce } from "lodash";
import { magentoInfos } from "./config"; // 为基础节点配置 
import { getAllTargetNodes, filterStr, splitEnter } from "@/utils";
// 核心是根据业务结构dom留存的数据并储存，可做优化，变成函数式的哦
export const lulalala = {
  subTotal: debounce(function() { // 获取总价分为包含税费和不包含税费
    let selectItem = document.querySelector(magentoInfos.subTotal);
    if (selectItem) {
      let nodes = getAllTargetNodes(selectItem, 1) // 子节点展平处理 =:)
        .filter(v => {
          let num = Number(filterStr(v.textContent || ""));
          return v.nodeName === "SPAN" && !isNaN(num);
        })
        .map(v => v.textContent)
        .sort(function(a, b) {
          return filterStr(b) - filterStr(a);
        });
      if (nodes.length) {
        localStorage.setItem("XHL_ORDER_SUBTOTAL", nodes[0]);
      }
      console.log("subtotal targetNodes", nodes);
    }
    console.log("subTotalTable is change", selectItem);
  }, 500),
  ship: debounce(function() {
    let selectItem = document.querySelector(magentoInfos.room.shippingContent);
    if (selectItem) {
      let arr = splitEnter(selectItem.textContent) // 截获的为纯文本处理成数组
        .filter(v => v)
        .map(v => v.trim())
        .filter(v => v);
      // 获取的标准的必填值所规定的的最低值格式
      localStorage.setItem(
        "XHL_CHECK_ADDRESS_SHIP",
        JSON.stringify({
          first_name: arr[0],
          last_name: arr[1],
          street_address: arr[2],
          city_state_province_postal_code: arr[3],
          country: arr[4],
          phone: arr[5]
        })
      );
    }
    console.log("shippingContent is change", selectItem);
  }, 500),
  payment: debounce(function() {
    // 业务逻辑
  }, 500)
}
// 注： config包含节点配置，如需改动或结构变动则需及时更新节点并做降级处理
```

## 总结

因为做的时候时间非常紧连带联调只有不到2天就匆匆上线，代码质量上有很多不足，后来是因为惰性也没有细致优化，想晒出来被大家鞭挞，在技术上越是蹂躏我，我就越兴奋，哈哈哈。。。咳咳，这边文章的初衷就是在进阶的路上路漫漫，好多有趣的API在等着我们去玩耍，新的技术也离不开原生api和结构的设计模式，故想慢慢总结将来想把react,vue源码的设计思想跟大家一起碰撞一下，欢迎指点哦~~~~
