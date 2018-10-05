---
published: false
---
Навеяно дискуссиями в группе...

*Саша*. В большинстве источников, которые я читал, паттерн Мост
рассматривается на примере нескольких операционных систем и API-интерфейсов рисования для
этих операционных систем. В других источниках паттерн рассматривается в контексте
рисования геометрических фигур палитрой цветов.

*Паша*. На редкость полезные примеры.

*Саша*. Как думаете, не считая операционных систем и геометрических фигур, паттерн находит
применение где-то еще?

*Маша*. Можем рассмотреть пример приложения для расчета заработной платы.

*Саша*. Где же там Мост увидим?

*Паша*. Спроектируем.

*Маша*. Начнем с того, что у нас есть тип Employee, которому надо начислить заработную плату.
А также есть несколько видов оплаты. Например, сдельная и повременная.

*Саша*. Можно сделать абстрактный тип PaymentType, у него будут деривативы -
Salaried и Commissioned. Employee будет содержать ссылку на абстрактный PaymentType и
делегировать ему работу по расчету заработной платы. Где же здесь Мост?

*Паша*. Моста нет, но похоже есть Стратегия.

*Маша*. Действительно, Стратегия есть. Тогда давайте представим, что Employee также может
получать бонусы к зарплате по разной схеме. Скажем, СashBonus и NonCashBonus.

*Саша*. Тогда у нас появится еще одна иерархия - PaymentBonus.

*Паша*. Или еще одна Стратегия.

*Саша*. Employee может запросить у PaymentType сумму начислений, а далее запросить у
PaymentBonus сумму бонусов.

*Саша* Сколько иерархий Стратегий не добавляй, все равно выглядит как Стратегия.

*Маша*. А еще выглядит как нарушение принципа Tell-Don't-Ask.

*Саша*. И чем же наши Стратегии его нарушают?

*Петя*. Видимо, вместо того, чтобы запрашивать данные у объекта, мы должны сказать,
что сделать с объектом. That is all OOP.

*Саша*. Тогда, чтобы Employee не запрашивал данные, а раздавал указания, можно
создать тип Paycheck, в который каждый тип Стратегии будет добавлять свои расчеты.

*Паша*. Тогда и пригодится Мост. Между Стратегиям. Заодно Шаблонный метод задействуем.

```c#
public abstract class PaycheckStation
{
    private PaycheckStation _next;

    public void Handle(Paycheck paycheck)
    {
        if (DoPaycheck(paycheck))
            _next?.Handle(paycheck);
    }

    public PaycheckStation SetNext(PaycheckStation next)
    {
        _next = next;
        return this;
    }

    protected abstract bool DoPaycheck(Paycheck paycheck);
}

public abstract class PaymentType : PaycheckStation
{
}

public class Salaried : PaymentType
{
    protected override bool DoPaycheck(Paycheck paycheck)
    {
        //some paycheck handling
        return true;
    }
}

public class Hourly : PaymentType
{
    protected override bool DoPaycheck(Paycheck paycheck)
    {
        //some paycheck handling
        return true;
    }
}

public abstract class Bonus : PaycheckStation
{
}

public class CashBonus : Bonus
{
    protected override bool DoPaycheck(Paycheck paycheck)
    {
        //some paycheck handling
        return true;
    }
}

public class Employee
{
    private PaymentType _type;
    private Bonus _bonus;
    private Deductor _deductor;

    public void SetType(PaymentType type) => _type = type;
    public void SetBonus(Bonus bonus) => _bonus = bonus;
    public void SetDeductor(Deductor deductor) => _deductor = deductor;

    public PaycheckStation GetBonus()
    {
        _bonus.SetNext(null);
        return _bonus;
    }

    public PaycheckStation GetDeductionAndBridgeTo(PaycheckStation next)
    {
        _deductor.SetNext(next);
        return _deductor;
    }
    public PaycheckStation GetPaymentAndBridgeTo(PaycheckStation next)
    {
        _type.SetNext(next);
        return _type;
    }
}

PaycheckStation start = employee.GetPaymentAndBridgeTo(
                             employee.GetBonusAndBridgeTo(
                               employee.GetDeductionAndBridgeTo(null)));
start.Handle(new Paycheck(employee));
```


*Саша*. Получается, только Employee знает о своей конфигурации, а Мост работает на
уровне абстракций. Ни одна из абстракций не знает обо всех остальных. Помещай любые абстракции,
в любом порядке.

*Паша*. А кто будет ответственный за создание деривативов PaycheckStation? Задействуем фабрику?

*Петя*. Что бы ты не задействовал, ясно одно. Кто создает Employee, тот создает
его при помощи этих деривативов, а значит тесно с ними связан.

*Паша*. Как и сам Employee тоже тесно связан с деривативами. Он знает, сколько есть
разных станций, а также знает их типы. Tight-coupling.

*Даша*. А Пол говорил, что нужно, чтобы классы имели слабую зависимость от других классов,
то есть не зависили от внешних изменений.

*Саша*. Значит, инвертируем зависимости.

*Маша*. Как вариант, вместо того, чтобы загружать Employee конкретными PaycheckStation, можно дать
возможность станциям самим определить, следует ли применить ее к конкретному Employee.

*Саша*. Но мы же пока определили у сотрудника три станции, а тут его нужно будет
прокатить через каждую станцию каждой иерархии?

*Витя*. Каждая из этих станций может очень быстро определить, нужно ли ей обработать запрос,
или нет. А если нет, она просто перенаправит запрос на следующую станцию. Low-coupling.

```c#
public abstract class PaycheckStation
{
    private PaycheckStation _next;

    public void Handle(Paycheck paycheck)
    {
      if (DoPaycheck(paycheck))
          _next?.Handle(paycheck);
    }

    public PaycheckStation SetNext(PaycheckStation next)
    {
        _next = next;
        return this;
    }

    protected abstract bool DoPaycheck(Paycheck paycheck);
}

public class SalariedCalculator : PaycheckStation
{
    public SalariedCalculator(PaycheckStation next) : base(next)
    {
    }

    protected override bool DoPaycheck(Paycheck paycheck)
    {
        if (paycheck.Employee.Type == Employee.PayType.Salaried)
        {
            //do some paycheck handling
        }
        return true;
    }
}

public class Employee
{
    public enum PayType { Salaried, Hourly }
    public enum PayDeduction { Taxes, Forfeit }
    public enum PayBonus { Cash, NonCash }

    public PayType Type { get; }
    public PayDeduction Deduction { get; }
    public PayBonus Bonus { get; }
}

public class Payroll
{
    private readonly IEnumerable<Employee> _employees;
    public PaycheckStation PaycheckStations { get; set; }

    public Payroll(IEnumerable<Employee> employees)
    {
        _employees = employees;
    }

    public void SetupPaycheckStations()
    {
            PaycheckStations = new SalariedCalculator(
                                 new HourlyCalculator(
                                    new CashBonus(
                                        new ForfeitDeductor(
                                            new TaxesDeductor(
                                                new ...(null)
                                            )
                                        )
                                    )
                                );

    }

    public void Calculate()
    {
        foreach (var employee in _employees)
        {
            PaycheckStations.HandlePaycheck(new Paycheck(employee));
        }
    }
}
```

*Паша*. Да какой там low-coupling, ведь при добавлении нового дериватива, ты должен
перекомпилировать и задеплоить всего Employee.

*Витя*. А при помощи Моста мы могли добавлять новые деривативы, не меняя ничего,
кроме фабрики, которая создает объект Employee.

*Саша*. Выходит, мы не избежим проблемы сцепления, какой бы подход ни выбрали.

*Даша*. Пол бы сказал, чтобы решить любую проблему в приложении, нужно добавить еще один
уровень абстракции.

*Петя*. Как бы нам в погоне за решением проблемы не придумать собственный
предметно-ориентированный язык.

*Маша*. DSL иногда действительно является лучшим способом решения сложной проблемы.

*Саша*. Как студент, я бы все-таки не взял на себя отетственность за выведению уровня абстракции
за рамки обычного программирования.

*Петя*. Боюсь, такое решение может преследоать тебя до конца твоих дней.

