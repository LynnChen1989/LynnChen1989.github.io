---
layout: post
title: Go面向对象
subtitle: 接口的理解
categories: Golang开发
tags: [goDev]
---

# interface接口的定义

  + 1.interface 是一组方法声明的集合。
  + 2.任何类型的对象实现了在interface 接口中声明的全部方法，则表明该类型实现了该接口。
  + 3.interface 可以作为一种数据类型，实现了该接口的任何对象都可以给对应的接口类型变量赋值。
  + 4.interface 可以被任意对象实现，一个类型/对象也可以实现多个 interface。
  + 5.接口中不能存在任何变量。
  + 6.一个接口（比如A接口）可以继承多个接口，这时如果要实现A接口，也必须将B，C接口的方法也全部实现。

```golang
package main

import "fmt"

type Phone interface {
    call()
}

type NokiaPhone struct {
}

func (nokiaPhone NokiaPhone) call() {
    fmt.Println("I am Nokia, I can call you!")
}

type ApplePhone struct {
}

func (iPhone ApplePhone) call() {
    fmt.Println("I am Apple Phone, I can call you!")
}

func main() {
    var phone Phone
    phone = new(NokiaPhone)
    phone.call()

    phone = new(ApplePhone)
    phone.call()
}

```
上述中体现了interface接口的语法，在main函数中，也体现了多态的特性。
同样一个phone的抽象接口，分别指向不同的实体对象，调用的call()方法，打印的效果不同，那么就是体现出了多态的特性。

# 接口的作用

那么接口的意义最终在哪里呢，实际上接口的最大的意义就是实现多态的思想，就是我们可以根据interface类型来设计API接口，那么这种API接口的适应能力不仅能适应当下所实现的全部模块，也适应未来实现的模块来进行调用。 调用未来可能就是接口的最大意义所在吧。

# 一个贷款业务设计例子

## 需求框架
```
贷款
  - 个人贷款
    - 个人消费贷
    - 个人抵押贷
  - 企业贷款
    - 经营性贷款
```

我们贷款先按照服务主体分为个人贷款和企业贷款，个人贷款又有个人消费贷款和个人抵押贷款。那么我们在服务设计的时候把系统分成3个模块层次，抽象层、实现层、业务逻辑层。

## 代码设计

[//]: # (![]&#40;../assets/images/golang/贷款抽象.png&#41;)
<img src="/assets/images/golang/贷款抽象.png" width="100%" height="100%" alt="" />
那么，我们首先将抽象层的模块和接口定义出来，这里就需要了interface接口的设计，然后我们依照抽象层，依次实现每个实现层的模块，在我们写实现层代码的时候，实际上我们只需要参考对应的抽象层实现就好了，实现每个模块，也和其他的实现的模块没有关系，这样也符合了上面介绍的开闭原则。这样实现起来每个模块只依赖对象的接口，而和其他模块没关系，依赖关系单一。系统容易扩展和维护。

我们在指定业务逻辑也是一样，只需要参考抽象层的接口来业务就好了，抽象层暴露出来的接口就是我们业务层可以使用的方法，然后可以通过多态的线下，接口指针指向哪个实现模块，调用了就是具体的实现方法，这样我们业务逻辑层也是依赖抽象成编程。我们就将这种的设计原则叫做`依赖倒置原则`。

## 完整代码

```golang
package main

import "fmt"

// --------------------------抽象层------------------------

// 贷款

type Loan interface {
	BorrowMoney()
	PaybackMoney()
}

// 借款人

type Loaner interface {
	DoLoanBusiness(loan Loan)
}

// --------------------------实现层------------------------

// 个人贷款

type PersonalLoan struct {
	name   string // 姓名
	amount int16  // 借款金额
}

func (pl *PersonalLoan) BorrowMoney() {
	fmt.Printf("个人贷款: \n"+
		"\t姓名: %s\n"+
		"\t借款: $ %d 元.\n", pl.name, pl.amount)
}

func (pl *PersonalLoan) PaybackMoney() {
	fmt.Printf("个人贷款:\n "+
		"\t姓名: %s\n"+
		"\t还款: $ %d 元.\n", pl.name, pl.amount)
}

//  个人消费贷款

type PersonalConsumeLoan struct {
	pl             PersonalLoan
	PersonalCredit string // 个人信用
}

func (pcl *PersonalConsumeLoan) BorrowMoney() {
	fmt.Printf("个人消费贷款: \n"+
		"\t姓名: %s \n"+
		"\t征信: %s \n"+
		"\t借款: $ %d 元.\n", pcl.pl.name, pcl.PersonalCredit, pcl.pl.amount)
}

func (pcl *PersonalConsumeLoan) PaybackMoney() {
	fmt.Printf("个人消费贷款: \n"+
		"\t姓名: %s \n"+
		"\t征信: %s \n"+
		"\t还款: $ %d 元.\n", pcl.pl.name, pcl.PersonalCredit, pcl.pl.amount)
}

func NewPersonalConsumeLoan(pl PersonalLoan, personalCredit string) *PersonalConsumeLoan {
	return &PersonalConsumeLoan{
		pl:             pl,
		PersonalCredit: personalCredit,
	}
}

// 个人抵押贷款

type PersonalMortgageLoan struct {
	pl             PersonalLoan
	MortgageAssess string //抵押物评估
}

func (pml *PersonalMortgageLoan) BorrowMoney() {
	fmt.Printf("个人抵押贷款: \n"+
		"\t姓名: %s \n"+
		"\t抵押物评估结果: %s \n"+
		"\t借款: $ %d 元.\n", pml.pl.name, pml.MortgageAssess, pml.pl.amount)
}

func (pml *PersonalMortgageLoan) PaybackMoney() {
	fmt.Printf("个人抵押贷款: \n"+
		"\t姓名: %s \n"+
		"\t抵押物评估: %s \n"+
		"\t还款: $ %d 元.\n", pml.pl.name, pml.MortgageAssess, pml.pl.amount)
}

// 企业贷款

type FirmLoan struct {
	name     string
	firmName string
	amount   int32
}

func (fl *FirmLoan) BorrowMoney() {
	fmt.Printf("企业经营贷款: \n"+
		"\t姓名: %s \n"+
		"\t公司名: %s \n"+
		"\t借款: $ %d 元.\n", fl.name, fl.firmName, fl.amount)
}

func (fl *FirmLoan) PaybackMoney() {
	fmt.Printf("企业经营贷款: \n"+
		"\t姓名: %s \n"+
		"\t公司名: %s \n"+
		"\t还款款: $ %d 元.\n", fl.name, fl.firmName, fl.amount)
}

func NewFirmLoan(name string, firmName string, amount int32) *FirmLoan {
	return &FirmLoan{name: name, firmName: firmName, amount: amount}
}

// 四川人办理业务

type SichuanPerson struct {
}

func (sp *SichuanPerson) DoLoanBusiness(loan Loan) {
	fmt.Println(" --- 四川人办理业务 ---")
	loan.BorrowMoney()

}

// --------------------------业务逻辑层-------------------------

// 四川人办理个人消费贷款
func sichuanPersonBorrowConsumeLoan() {
	var personConsumeLoan Loan
	personConsumeLoan = NewPersonalConsumeLoan(
		PersonalLoan{name: "snake chen", amount: 10000},
		"个人征信结果 [good!]")
	var scp Loaner
	scp = &SichuanPerson{}
	scp.DoLoanBusiness(personConsumeLoan)
}

// 四川人办理公司经营贷款
func sichuanPersonBorrowFirmLoan() {
	var firmLoan Loan
	firmLoan = NewFirmLoan(
		"snake chen",
		"牛逼闪闪公司",
		10000000,
	)
	var scp Loaner
	scp = &SichuanPerson{}
	scp.DoLoanBusiness(firmLoan)
}

//  --------------------------业务入口-------------------------
func main() {
	sichuanPersonBorrowConsumeLoan()
	sichuanPersonBorrowFirmLoan()
}

```

## UML图
<img src="/assets/images/golang/boss.png" width="100%" height="100%" alt="" />

[//]: # (![]&#40;../assets/images/golang/boss.png&#41;)

从代码和UML图可以看出，我们定义了两个接口Loan和Loaner，Loan接口包括两个抽象方法BorrowMoney()和PaybackMoney()方法。
然后我们定义了两个基类FirmLoan和PersonalLoan，PersonalConsumeLoan和PersonalMortgageLoan集成PersonalLoan，为PersonalLoan，PersonalConsumeLoan，PersonalMortgageLoan，FirmLoan
四个类都实现了BorrowMoney()和PaybackMoney()方法，也就是说这四个类实现了Loan这个接口。
