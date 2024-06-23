# 从Cypress到PlayWright

## 背景

在项目的Board上看到了一张需要添加端到端测试的卡。

当同样的东西在生活中出现两次的时候，我们就需要把它~~（抽出来）~~，插入思考列表了。

## 

## 1. 端到端测试

### 1.1 端到端测试

在测试金字塔中，端到端测试位于很高的层级。它从头到尾测试来整个项目以确保产品流按预期运行。它的主要目的是通过模拟真实用户场景并验证被测系统及其组件的集成和数据完整性，从最终用户的体验进行测试。

从Mattin Fowler的测试层级图中也可以看到，端到端测试是处于最外围的测试，保证了程序的运行。



### 1.2. 端到端测试框架

那E2E测试的框架，在我看来最重要的有两个方面：

第一个是能不能真实还原用户使用场景，因为这是我们端到端测试的根本目的。

第二个是这个运行测试的速度是不是足够快，因为这决定了我们的测试性能和体验。

接下来就真刀真枪地对比一下Cypress和Playwright

## 2. Cypress VS Playwright

| 我体验过并觉得重要的区别 | Cypress           | Playwright                              |
| ------------------------ | ----------------- | --------------------------------------- |
| 支持的内核               | Chromium、Firefox | Chromium、Firefox 和 Webkit (Safari)    |
| 支持的语言               | JavaScript        | JavaScript/TypeScript, Java, C#, Python |
| 天然支持并发执行         | 不可              | 简单易用                                |
| 自动录制脚本             | 支持              | 支持，有注释                            |

### 2.1 支持的内核

相比之下，Playwright多支持了Webkit内核的浏览器。

这一点重要的原因是如果不关注在不同浏览器上的表现，使用Webkit内核运行测试的速度比Chromium更快。

此外，如果需要关注在webkit上浏览器的表现的话，使用Cypress就无法满足需求了。

### 2.2 支持的语言

语言再也不是不学习的理由啦，Java, Python, C#都是你打开Playwright大门的钥匙。

甚至同样在Playwright中，个人觉得Playwright的API的语法要优于Cypress，尤其是断言部分。

### 2.3 并行

这是我认为PlayWright**最强大**的部分。

- 默认情况下，**多个测试文件**是并行运行的。单个文件中的测试在同一个工作进程中按顺序运行。

- 默认情况下，**单个测试文件**中的测试按顺序运行。如果单个文件中有许多**独立测试**，可以像下面这样并行运行它们。

  ```javascript
  import { test } from '@playwright/test';
  
  test.describe.parallel('suite', () => {
    test('runs in parallel 1', async ({ page }) => { /* ... */ });
    test('runs in parallel 2', async ({ page }) => { /* ... */ });
  });
  ```

### 2.4 自动化脚本录制

这是我认为一个自动化测试框架**最实用**的部分。自动化脚本录制可以帮助我们从用户视角来书写测试，保证了端到端测试的意义。同时极大地加快了写测试的速度。

Cypress可以修改Cypress.json来开启CypressStudio，即自动化脚本录制功能。

```javascript
{
  "experimentalStudio": true
}
```

Playwright可以通过输入playwright codegen打开一个浏览器和代码生成器。



下面是我分别用二者录制一个用户注册页面的结果：

Cypress录制结果

```cypher
cy.visit('http://localhost:3000/sign-up');
cy.get('#email').type('Fawn520@example.com');
cy.get('#password').click();
cy.get('#password').type('Fawn520');
cy.get('#name').click();
cy.get('#name').type('New User');
cy.get('#phonenumber').click();
cy.get('#phonenumber').type('1234567890');
cy.get('#country-select').click();
cy.get('#country-select').type('UK');
cy.get('#iacceptthe').click();
cy.get('.button').click();
cy.get('.create-account').submit();
```

PlayWright录制结果

```javascript
test('basic test', async ({page}) => {
    await page.goto('http://localhost:3000/sign-up);
    // Click text=Sign Up
    await page.click('text=Sign Up');
    assert.equal(page.url(), 'http://localhost:3000/sign-up');

    // Click input[type="text"]
    await page.click('input[type="text"]');

    // Fill input[type="text"]
    await page.fill('input[type="text"]', 'Fawn520@example.com');

    // Click input[type="password"]
    await page.click('input[type="password"]');

    // Fill input[type="password"]
    await page.fill('input[type="password"]', 'Fawn520');

    // Press Tab
    await page.press('input[type="password"]', 'Tab');

    // Click #name
    await page.click('#name');

    // Fill #name
    await page.fill('#name', 'New User');

    // Click input[type="tel"]
    await page.click('input[type="tel"]');

    // Fill input[type="tel"]
    await page.fill('input[type="tel"]', '123456789');

    // Select UK
    await page.selectOption('select', 'UK');

    // Check input[type="checkbox"]
    await page.check('input[type="checkbox"]');
});
```



这里三处我觉得值得关注

- 语法上的差异
- Playwright自动生成注释
- 选择器的不同选择方式

可以看出playwright的选择器的方式更偏向于从标签来选择元素，而有时Cypress更倾向于从组件类型来选择。从测试目的角度来看，我觉得从标签来选择更符合需求。因为用户点击一个注册按钮是因为这个按钮上面写着“Sign Up”，而不是因为他是一个按钮。

但是像Playwright这样的策略也会导致有时无法成功录制。比如我想录制一个下拉菜单，playwright甚至无法成功录制鼠标悬停这个操作。

Cypress录制成功

```javascript
cy.visit('http://localhost:3000/');
cy.get('.custom-dropdown__menu > a:nth-child(1)').click(); 
```

Playwright录制失败：

```Session反馈：
// Click text=Account
await page.click('text=Account');
```



Session反馈：



### 小结

看起来Playwright在很多方面都优于Cypress，且Playwright还有很多我没有来得及尝试和体验的功能比如测试分片，跨域测试等。但二者都是优秀的端到端测试框架，还是要结合具体的使用场景来选取更适合自己的技术。

如果说代码是开发者的剑，测试像是剑鞘。保护剑的同时也决定了剑锋所指何处。







