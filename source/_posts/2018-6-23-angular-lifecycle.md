---
title: Angular Component 生命周期钩子顺序
date: 2018-06-23
categories: angular
tags: [angular]
---
本文详细分析和说明了 Angular 生命周期钩子，并给出了使用建议。
<!-- more -->

作者：前端咖秀
链接：https://www.jianshu.com/p/4ac9994e0f23

本文详细分析和说明了 Angular 生命周期钩子，并给出了使用建议。
```
ngOnChanges
ngOnInit
DoCheck(3x)
AfterContentInit
AfterContentChecked(3x)
AfterViewInit
AfterViewChecked(3x)
OnDestroy
```
注意：DoCheck,AfterContentChecked和AfterViewChecked这三种钩子被触发的次数很多，我们必须要精简这三种钩子的逻辑。

### OnInit和OnDestroy
通过使用指令来发现一个元素什么时候被初始化或者被销毁。

* 就像对组件一样，Angular也会对指令调用这些钩子方法
* 使用指令可以使我们在不直接修改DOM实现代码的情况下，透视其内部细节
```typescript
import { Directive, OnInit, OnDestroy } from '@angular/core';

import { LoggerService } from './logger.service';

let nextId = 1;

// Spy on any element to which it is applied.
// Usage: <div mySpy>...</div>
@Directive({selector: '[mySpy]'})
export class SpyDirective implements OnInit, OnDestroy {

  constructor(private logger: LoggerService) { }

  ngOnInit()    { this.logIt(`onInit`); }

  ngOnDestroy() { this.logIt(`onDestroy`); }

  private logIt(msg: string) {
    this.logger.log(`Spy #${nextId++} ${msg}`);
  }
}
```
把这个指令写到任何原生元素或者组件元素上，它会与所在的组件同时初始化和销毁
```html
<div *ngFor="let hero of heroes" mySpy class="heroes">
  {{hero}}
</div>
```
* OnInit（）是组件获取初始数据最好的地方
* OnDestroy（）用来释放那些不会被垃圾收集器自动回收的各类资源的地方，如取消那些对可观察对象和DOM事件的订阅；停止定时器；注销该指令曾注册到全局服务或应用级服务中的各种回调函数；如果不这么做，会导致内存泄漏的风险。

### OnChanges()

一旦检测到该组件（或指令）的输入属性发生变化，Angular就会调用它的ngOnChanges（）方法
```typescript
@Input() hero: Hero;
@Input() power: string;

changeLog: string[] = [];

ngOnChanges(changes: SimpleChanges) {
    for (let propName in changes) {
      let chng = changes[propName];
      let cur  = JSON.stringify(chng.currentValue);
      let prev = JSON.stringify(chng.previousValue);
      this.changeLog.push(`${propName}: currentValue = ${cur}, previousValue = ${prev}`);
    }
}
<div class="parent">
  <h2>{{title}}</h2>

  <table>
    <tr><td>Power: </td><td><input [(ngModel)]="power"></td></tr>
    <tr><td>Hero.name: </td><td><input [(ngModel)]="hero.name"></td></tr>
  </table>
  <p><button (click)="reset()">Reset Log</button></p>

  <on-changes [hero]="hero" [power]="power"></on-changes>
</div>
```
* 当power属性的字符串值变化时，相应的日志就出现了
* 但是，ngOnChanges并没有捕捉到hero.name的变化，Angular只会在输入属性的值变化时调用这个钩子，而hero属性的值是一个到hero对象的引用，Angular不会关注这个hero对象的name属性的变化，这个hero对象的引用并没有发生变化，于是，Angular不会报告什么变化。

### DoCheck
使用DoCheck来检测Angular自身无法捕获的变更并采取行动
*ngDoCheck()会被非常频繁的调用，需要使用轻量级的逻辑，否则会损害用户体验*  

通过捕获当前值和旧值进行比较
```typescript
//代码片段:
changeDetected = false;
changeLog: string[] = [];
oldHeroName = '';
oldPower = '';
oldLogLength = 0;
noChangeCount = 0;

ngDoCheck() {
    if (this.hero.name !== this.oldHeroName) {
      this.changeDetected = true;
      this.changeLog.push(`DoCheck: Hero name changed to "${this.hero.name}" from "${this.oldHeroName}"`);
      this.oldHeroName = this.hero.name;
    }

    if (this.power !== this.oldPower) {
      this.changeDetected = true;
      this.changeLog.push(`DoCheck: Power changed to "${this.power}" from "${this.oldPower}"`);
      this.oldPower = this.power;
    }

    if (this.changeDetected) {
        this.noChangeCount = 0;
    } else {
        // log that hook was called when there was no relevant change.
        let count = this.noChangeCount += 1;
        let noChangeMsg = `DoCheck called ${count}x when no change to hero or power`;
        if (count === 1) {
          // add new "no change" message
          this.changeLog.push(noChangeMsg);
        } else {
          // update last "no change" message
          this.changeLog[this.changeLog.length - 1] = noChangeMsg;
        }
    }

    this.changeDetected = false;
 }
```
### AfterView
Angular会在每次创建来子视图之后调用AfterViewInit()和AfterViewChecked()钩子
```typescript
import { AfterViewChecked, AfterViewInit, Component, ViewChild } from '@angular/core';

import { LoggerService }  from './logger.service';

//////////////////
@Component({
  selector: 'app-child-view',
  template: '<input [(ngModel)]="hero">'
})
export class ChildViewComponent {
  hero = 'Magneta';
}

//////////////////////
@Component({
  selector: 'after-view',
  template: `
    <div>-- child view begins --</div>
      <app-child-view></app-child-view>
    <div>-- child view ends --</div>`
   + `
    <p *ngIf="comment" class="comment">
      {{comment}}
    </p>
  `
})
export class AfterViewComponent implements  AfterViewChecked, AfterViewInit {
  private prevHero = '';

  // Query for a VIEW child of type `ChildViewComponent`
  @ViewChild(ChildViewComponent) viewChild: ChildViewComponent;

  constructor(private logger: LoggerService) {
    this.logIt('AfterView constructor');
  }

  ngAfterViewInit() {
    // viewChild is set after the view has been initialized
    this.logIt('AfterViewInit');
    this.doSomething();
  }

  ngAfterViewChecked() {
    // viewChild is updated after the view has been checked
    if (this.prevHero === this.viewChild.hero) {
      this.logIt('AfterViewChecked (no change)');
    } else {
      this.prevHero = this.viewChild.hero;
      this.logIt('AfterViewChecked');
      this.doSomething();
    }
  }

  comment = '';

  // This surrogate for real business logic sets the `comment`
  private doSomething() {
    let c = this.viewChild.hero.length > 10 ? `That's a long name` : '';
    if (c !== this.comment) {
      // Wait a tick because the component's view has already been checked
      this.logger.tick_then(() => this.comment = c);
    }
  }

  private logIt(method: string) {
    let child = this.viewChild;
    let message = `${method}: ${child ? child.hero : 'no'} child view`;
    this.logger.log(message);
  }
  // ...
}

//////////////
@Component({
  selector: 'after-view-parent',
  template: `
  <div class="parent">
    <h2>AfterView</h2>

    <after-view  *ngIf="show"></after-view>

    <h4>-- AfterView Logs --</h4>
    <p><button (click)="reset()">Reset</button></p>
    <div *ngFor="let msg of logs">{{msg}}</div>
  </div>
  `,
  styles: ['.parent {background: burlywood}'],
  providers: [LoggerService]
})
export class AfterViewParentComponent {
  logs: string[];
  show = true;

  constructor(private logger: LoggerService) {
    this.logs = logger.logs;
  }

  reset() {
    this.logs.length = 0;
    // quickly remove and reload AfterViewComponent which recreates it
    this.show = false;
    this.logger.tick_then(() => this.show = true);
  }
}
```

在更新comment之前，doSomething()方法都要等上一拍（tick）

* Angular的单向数据流规则禁止在一个视图已经被组合好之后再更新视图，而这两个钩子都是在组件的视图已经被组合好后触发的，如果我们立即更新组件中被绑定的comment属性，Angular就会抛出一个错误，LoggerService.tick_then()方法延迟更新日志的一个回合（浏览器javascript周期回合）
* Angular会频繁调用AfterViewChecked,需要注意精简代码

### AfterContent
Angular会在外来内容被投影到组件中之后调用 AfterContentInit() 和 AfterContentChecked() 钩子
内容投影 是从组件外部导入HTML内容，并把它插入在组件模版中指定的位置上的一种途径
//父组件模版
```html
`<after-content>
   <app-child></app-child>
 </after-content>`
```
<app-child>被包含在<after-content>标签中，永远不要在组件标签的内部放任何内容－－除非我们想把这些内容投影进这个组件中

//<after-content>组件模版
```html
template: `
  <div>-- projected content begins --</div>
    <ng-content></ng-content>
  <div>-- projected content ends --</div>`
```
<ng-content>标签是外来内容的占位符，它告诉Angular在哪里插入这些外来的内容，在这里，被投影进的内容就是来自父组件的<app-child>标签

### AfterView和AfterContent的不同点

AfterView钩子关心的是ViewChildren,这些子组件的元素标签会出现在该组件的模版里面
AfterContent钩子关心的是ContentChildren,这些子组件被Angular投影进该组件中
使用AfterContent不用担心单向数据流规则
Angular在每次调用AfterView钩子之前也会同时调用AfterContent,Angular在完成当前组件的视图合成之前，就已经合成了被投影内容的合成，所以仍然有机会修改视图
