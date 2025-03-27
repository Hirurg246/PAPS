# Лабораторная работа №3
## Диаграмма контейнеров

## Диаграмма компонентов
### Приложение на терминале

## Диаграмма последовательностей
Выберем use case "Осуществить доступ к системам умного здания". На диаграмме последовательности раскроем контейнер "Приложение на терминале", а все остальные опишем как неделимые.

В диаграмме показан процесс получения доступа пользователем к системе умного здания. Каждая передача сообщения между модулями системы, начиная с обработчика запросов сопровождается операциями шифровки/расшифровки данных сообщения.
## Модель хранилища данных

Хранилище данных состоит из базы данных в 3НФ и участка файловой системы сервера. БД хранит сущности предметной области, а файловая система - большие файлы, ссылки на которые хранятся в БД. Модель сделана максимально изолированной, так как разрабатывается как часть системы безопасности.
## Применение основных принципов разработки
### KISS (Keep It Simple, Stupid)
Код должен делать ровно то что от него ожидается, быть простым и понятным. Класс ниже управляет существованием терминалов, позволяя создавать и удалять их и ничего более.
```csharp
static class TerminalManager
{
    public static IndexedHashSet<Terminal> Terminals = [];
    public static Terminal CreateTerminal(Room room)
    {
        Terminal terminal = new Terminal(Terminals.GetNextId());
        Terminals.Add(terminal);
        room.AddTerminal(terminal);
        return terminal;
    }

    public static int DeleteTerminal(int id)
    {
        Terminal terminal = Terminals.PopById(id);
        if (terminal is not null)
        {
            terminal.Room.RemoveTerminal(terminal);
            return 1;
        }
        return 0;
    }
}
```
### YAGNI (You Ain’t Gonna Need It)
Реализовывать только то, что нужно текущей итерации проекта, не забегать вперёд. Класс ниже может быть расширен, если бы я реализовывал удалённое управление терминалами. Так как такой задачи пока не стоит - заготовки методов управления отсутствуют и класс содержит только описание физического объекта и его переносимость между комнатами.
```charp
class Terminal: IMoveable
{
    public int Id;
    public Room Room;
    public int Status;

    public Terminal(int id)
    {
        Id = id;
        Status = 0;
    }
    public void Move(Room newRoom)
    {
        Room = newRoom;
    }
}
```
### DRY (Don't Repeat Yourself)
Стараться писать максимально переиспользуемый код. Вынесение повторяющегося кода в отдельную функцию воплощает данный принцип. В данном случае мы заменили 6 повторяющихся строк кода двумя вызовами функции.
```csharp
Terminal terminal1 = TerminalManager.CreateTerminal(room1);
Terminal terminal2 = TerminalManager.CreateTerminal(room2);
```
### SOLID
#### S - Single Responsibility Principle (Принцип единственной ответственности)
Каждый объект кода должен иметь лишь одну функцию или ответственность. Не нужно пытаться создать суперкласс реализующий все функции одновременно. Как уже было сказано, класс ниже управляет существованием терминалов, позволяя создавать и удалять их и ничего более.
```csharp
static class TerminalManager
{
    public static IndexedHashSet<Terminal> Terminals = [];
    public static Terminal CreateTerminal(Room room)
    {
        Terminal terminal = new Terminal(Terminals.GetNextId());
        Terminals.Add(terminal);
        room.AddTerminal(terminal);
        return terminal;
    }

    public static int DeleteTerminal(int id)
    {
        Terminal terminal = Terminals.PopById(id);
        if (terminal is not null)
        {
            terminal.Room.RemoveTerminal(terminal);
            return 1;
        }
        return 0;
    }
}
```
#### O - Open/Closed Principle (Принцип открытости/закрытости)
Классы открыты для расширения, но закрыты для модификации. Дочерние классы добавляет свой специфический функционал, но не модифицирует базовое поведение класса. В примере объекты дочерних классов получают свой уникальный функционал, но всё ещё должны выдавать журнал доступа к ним
```csharp
abstract class SecurityObject
{
    protected int id;
    protected Room room;

    public SecurityObject(int id, Room room)
    {
        this.id = id;
        this.room = room;
    }

    public abstract AccessJournal GetAccessJournal();
}

abstract class AccessJournal
{...}

class LockAccessJournal: AccessJournal
{...}

class CameraAccessJournal : AccessJournal
{...}

class Lock: SecurityObject
{
    protected Room nextRoom;

    public Lock(int id, Room room, Room nextRoom) : base(id, room)
    {
        this.nextRoom = nextRoom;
    }

    public void Open()
    {
        //Открыть замок
    }

    public void Close()
    {
        //Закрыть замок
    }

    public override LockAccessJournal GetAccessJournal()
    {
        //Получить журнал доступа к замку
    }
}

class Camera : SecurityObject, IMoveable
{
    protected Position position;

    public Camera(int id, Room room) : base(id, room)
    {
        this.position = Position.StartingPosition;
    }

    public void ChangePosition(Position newPosition)
    {
        //Меняем положение
        this.position = newPosition;
    }

    public override CameraAccessJournal GetAccessJournal()
    {
        //Получить журнал доступа
    }
}
```
#### L - Liskov Substitution Principle (Принцип подстановки Лисков)
Объекты дочерних классов должны быть взаимозаменяемыми с объектами родительского класса без изменения правильности программы. Реализация OCP даёт возможность сгруппировать объекты разных классов в один список и вызывать для каждого из них функции общего родительского класса. Новые подклассы также смогут войти в этот список без потери родительского функционала. В примере мы добавляем в архив различные виды журналов доступа от объектов дочерних классов.
```csharp
Camera camera1 = CameraManager.CreateCamera(room1);
Lock lock1 = LockManager.CreateLock(room1, room2);
List<SecurityObject> securityObjects = new List<SecurityObject> { camera1, lock1 };
//Прошло время
foreach (SecurityObject so in securityObjects) AccessArchive.Add(so.GetAccessJournal());
``` 
#### I - Interface Segregation Principle (Принцип разделения интерфейса)
Класс не должен иметь нереализованных интерфейсов, а только те которые ему действительно нужны. Тогда программист будет точно знать что объекты класса могут делать. В примере мы применяем интерфейс IMoveable к камере, но не замку, так как камеру можно перенести из комнаты в комнату, а замок зафиксирован в одной двери.
```csharp
interface IMoveable
{
    public void Move(Room newRoom);
}
class Camera : SecurityObject, IMoveable
{
    ...
    public void Move(Room newRoom)
    {
        room = newRoom;
    }
}
class Lock: SecurityObject
{...}
```
#### D - Dependency Inversion Principle (Принцип инверсии зависимостей)
Высокоуровневые модули не должны зависить от имплементации нижнеуровневых модулей. Они должны зависеть и работать от абстракций, оставляя реализацию и прочие детали за скобками. В примере AccessJournalCrossAnalyzer не важно какие типы объектов системы безопасности поступили в него. Перекрёстный анализ пользователей состоится для любых дочерних классов SecurityObject.
```csharp
Camera camera1 = CameraManager.CreateCamera(room1);
Lock lock1 = LockManager.CreateLock(room1, room2);
//Прошло время
AccessJournalCrossAnalyzer ajAnalyzer = new AccessJournalCrossAnalyzer { camera1, lock1 };
var result = ajAnalyzer.CrossAnalyze();
```
## Дополнительные принципы разработки
### BDUF. Big design up front («Масштабное проектирование прежде всего»)
**BDUF** - принцип, предлагающий "думать перед тем как прыгать", а именно планировать разработку в деталях до начала непосредственного написания кода. Лично я пользовался этим принципом с первого курса, когда мой научный руководитель сказал "Убери свой код, просто покажи что он должен делать". В личных проектах как этот применение принципа очень удобно, так как разработка ведётся одним человеком от начала до конца и сложенная в голове концепция может быть полностью перенесена "на бумагу". Писать код после построения архитектуры всё равно что заполнять детскую раскраску.
Принцип BDUF в разработке я **использую**.
### SoC. Separation оf concerns (Принцип разделения ответственности)
**SoC** - принцип, предлагающий разбить систему на логические компоненты, которые можно реализовать и протестировать независимо. Учитывая выбор архитектуры "модульный монолит", система и так разбивается на связанные, но независимые модули, реализуя данный принцип. В дополнение, система безопасности никак не может опираться на систему, которая "умирает вся и сразу".
Принцип SoC в разработке я **использую**.
### MVP. Minimum viable product (Минимально жизнеспособный продукт)
**MVP** - проект, имеющий лишь только базовый функционал, но при этом имеющий ценность для пользователя. Таким образом, его уже можно развернуть на практике, если не думать о возможных последствиях. Учитывая слабую протестированность и отсутствие крутых фич, система будет падать, а пользователи вопрошать зачем они её установили. Тем не менее, в требованиях к моему диплому значится разработка именно MVP, так что отделаться описанным ниже PoC не получится. Желать большего также сложно, поскольку для моей системы сложно даже найти место полноценного развёртывания.
Принцип MVP в разработке я **использую**.
### PoC. Proof of concept (Доказательство концепции)
**PoC** - ситуация, когда рисунок ключа складывают в бумажный ключ. Итоговая система служит только для того чтобы показать, что время было проведено не зря, у системы есть шанс на реальную реализацию. Где-то далеко. По сути это даже не система, а набор разрозненных функций, работающих хоть через консоль. Личные проекты часто остаются в этой стадии из-за нехватки энтузиазма. Тем не менее дипломная работа требует большей серьёзности подхода, чем "это можно сделать (наверное, может нет)". 
Принцип PoC в разработке мы я **не использую**.
