# Неконстантные константы

В С++ есть ключевое слово `const`, позволяющее помечать значения как неизменяемые.
Также в C++ есть `const_cast`, позволяющий этот `const` игнорировать. И иногда за это вам ничего не будет.
А иногда будет неопределенное поведение, segfault, и прочие радости жизни.

Разница между этими "иногда" в том, что есть настоящие константы, попытка модификации которых -- UB.
А есть ссылки на константы, ссылающиеся не на константы. 
И раз на самом деле объект неконстантен, то модифицировать его можно без проблем.

Так, например, эту "фичу" можно [эксплуатировать](https://godbolt.org/z/oW4YeP), чтобы не повторять один и тот же код для `const` и не-`const` методов класса
```C++
class MyMap {
public:
    // какой-то метод с длинной реализацией:
    const int& get_for_val_or_abs_val(int val) const {
        const auto it_val = m.find(val);
        if (it_val != m.end()) {
            return it_val->second;
        }
        const auto abs_val = std::abs(val);
        const auto it_abs = m.find(abs_val);
        if (it_abs != m.end()) {
            return it_abs->second;
        }
        throw std::runtime_error("no value");
    }

    int& get_for_val_or_abs_val(int val) {
        return const_cast<int&>( // отбрасываем const с результата. 
            // находясь в неконстантном методе, мы знаем, что результат 
            // в действительности не является константой -- проблем не будет.
            std::as_const(*this) // навешиваем const, чтобы вызвать const-метод,
            // а не уйти в бесконечную рекурсию
            .get_for_val_or_abs_val(val));
    }

    void set_val(int val, int x) {
        m[val] = x;
    }
private:
    std::map<int, int> m;
};
```

По возможности, стоит избегать такого кода. Видно, что он очень хрупок -- забытый или случайно удаленный
`std::as_const` ломает его. И без настройки предупреждений, компиляторы об этом сообщать [не торопятся](https://godbolt.org/z/17fc31).

Вместо использования `const_cast` и привнесения в мир C++ еще большей нестабильности, 
решить проблему диблирования кода можно с помощью шаблонного метода:
```C++
class MyMap {
public:
    // какой-то метод с длинной реализацией:
    const int& get_for_val_or_abs_val(int val) const {
        return get_for_val_or_abs_val_impl(*this, val); // *this -- const&
    }

    int& get_for_val_or_abs_val(int val) {
        return get_for_val_or_abs_val_impl(*this, val); // *this  -- &
    }

    void set_val(int val, int x) {
        m[val] = x;
    }
private:
    template <class Self>
    static decltype(auto) get_for_val_or_abs_val_impl(Self& self, int val) {
        auto&& m = self.m;
        if (it_val != m.end()) {
            return (it_val->second); // допскобки для вывода категории значения
        }
        const auto abs_val = std::abs(val);
        const auto it_abs = m.find(abs_val);
        if (it_abs != m.end()) {
            return (it_abs->second);
        }
        throw std::runtime_error("no value");
    }

    std::map<int, int> m;
};
```
У этого варианта есть свои недостатки, сломать его еще проще (скобки и `decltype`). Но, единожды его написав, можно рассчитывать на то, что странная магия отпугнет желающих этот код поправить.

Конечно, вместо `decltype(auto)` можно написать чуть больше кода, с явным укзанием типов возвращаемых значений.

## Сonst и оптимизации

Операции над иммутабельными данными отлично оптимизируются, распараллеливаются, и вообще ведут себя здорово.

Однако, возможность заигрывать со снятием и навешиванием `const` где угодно в коде, 
вообще говоря, исключает этот ряд оптимизаций. Так повторное обращение по константной ссылке
к одному и тому же полю или методу совсем не обязано кэшироваться.

И, например, итерирование по вектору не может быть [оптимизировано](https://godbolt.org/z/1fs4W3) в таком простом случае:

```C++
using predicate = bool (*) (int);

int count_if(const std::vector<int>& v, predicate p) {
     int res = 0;
     for (size_t i = 0; i < v.size(); ++i) { // значение v.size() нельзя 
                                             // единожды сохранить в регистре 
         if (p(v[i])) {  //конкретный p может иметь доступ к изменяемой ссылке на
                         // этот же самый v
             ++res;
         }
         // код метода size() придется выполнять на каждой итерации!
     }
     return res;
}
```

Пример, запрещающий оптимизацию, может быть не очевиден. Но тоже прост:
```C++
std::vector<int> global_v = {1};

bool pred(int x) {
    if (x == global_v.size()) {
        global_v.push_back(x);
        return true;
    } else {
        return false;
    }
}

int main() {
    return count_if(global_v, pred);
}
```

Этот код очень плох. Он не должен нигде встречаться; его никто не пропустит на ревью. Но так теоретически написать можно, поэтому оптимизация невыполняется.

Если в качестве типа предиката использовать шаблонный параметр, 
можно привести куда более изощренные примеры без привлечения глобальных переменных.

Учитывая ограниченные возможности на автоматическую оптимизацию, подобный цикл переписывают (делая ту самую работу, которую ждали от компилятора):
```C++
int count_if(const std::vector<int>& v, predicate p) {
    int res = 0;

    for (auto x : v) { // range-based-for не обращается к size()
                       // а один раз считывает begin/end указатели и работает 
                       // с ними; разность begin - end не рассчитывается
        if (p(v[i])) {  
             ++res;
        }
    }
    return res;
}
```
В таком случае, при передаче "нехорошего" предиката, меняющего вектор, мы получим неопределенное поведение. Но это уже совсем другая история...

Вот менее тривиальный пример `const`, никак не способствующего оптимизации:
```C++
void find_last_zero_pos(const std::vector<int>& v, 
                        const int* *poiner_to_last_zero) {
     *poiner_to_last_zero = 0;
     for (size_t i = 0; i < v.size(); ++i) { // мы опять не можем один раз 
                                             // сохранить значение v.size()
         if (v[i] == 0) {
             // внутри вектора есть поля типа int* -- begin, end
             // что если pointer_to_last_zero указывает на один из них?!?
             *poiner_to_last_zero = (v.data() + i);
         }
         // пересчитываем size!
     }
}
```

Оставаясь в рамках рекомендуемых практик написания C++ программ, 
мы не можем соорудить пример, который бы демонстрировал неприменимость оптимизации -- нам мешает инкапсуляция.
До приватных полей вектора мы не можем законно добраться.

Но ненормальный код не запрещен! Применим грубую силу:
```C++
int main() {
    std::vector<int> a = {1,2,4,0};
    const int* &data_ptr = reinterpret_cast<const int* &>(a); // ссылка на begin!
    find_last_zero_pos(a, &data_ptr);
}
```
И вот мы имеем парадоксальный результат: возможность написать явно некорректный код, запрещает компилятору оптимизировать цикл!
И вся концепция неопределенного поведения как возможности для оптимизации (некорректного кода не бывает) разваливается.

Что ж, по крайней мере, для этого примера имеется некоторая стабильность: 
Исходный цикл со счетчиком и переписанный на range-based-for закончатся на неопределенном поведении.

В Rust же, благодаря семантике владения, все эти циклы могут быть успешно оптимизированы.

## Сonst, время жизни и происхождение указателей. 

Неизменяемые объекты всем хороши, кроме одного -- это константные объекты в C++.
Если они где-то засели, то их оттуда по-нормальному не выгонишь.

Что имеется в виду:

Пусть есть структура с константным полем

```C++
struct Unit {
    const int id;
    int health;
};
```

Из-за константного поля объекты `Unit` теряют операцию присваивания. 
Их нелья менять местами -- `std::swap` не работает больше.
`std::vector<Unit>` нельзя больше просто так отсортировать... В общем, сплошное удобство.

Но самое интересное начинается, если сделать что-то такое
```C++
std::vector<Unit> units;
unit.emplace_back(Unit{1, 2});
std::cout << unit.back().id <<  " ";
unit.pop_back();
unit.emplace_back(Unit{2, 3});
std::cout << unit.back().id <<  "";
```

В зависимости от того, смогли ли при реализации вектора задушить агрессивные оптимизации компилятора, такой код может вывести
либо `1 2 ` (все хорошо), либо `1 1 ` (компилятор соптимизировал константное поле!)

Компилятор имеет право воспринимать происходящее следующим образом: 
- В векторе 1 элемент
- Вектор не реаллоцировался.
- Указатель на элемет в первом `cout` и во втором `cout` один и тот же.
- И там и там используется константное поле
- Я его уже читал при первом `cout`
- Зачем мне его читать еще раз, это же константа!
- Вывожу закэшированное значение.

К сожалению или к счастью, воспроизвести подобное поведение компилятора на практике не получается.
Тем не менее, вот такой код, который может использоваться для реализации самописных `std::optional`, 
по стандарту содержит UB (и не одно!)

```C++
    using storage = std::aligned_storage_t<sizeof(Unit), alignof(Unit)>;
    storage s;
    new (&s) Unit{1,2};
    std::cout << reinterpret_cast<Unit*>(&s)->id << "\n"; // UB
    reinterpret_cast<Unit*>(&s)->~Unit(); // UB
    new (&s) Unit{2,2};
    std::cout << reinterpret_cast<Unit*>(&s)->id << "\n"; // UB
    reinterpret_cast<Unit*>(&s)->~Unit(); // UB
```

Правильный же вариант:
```C++
    using storage = std::aligned_storage_t<sizeof(Unit), alignof(Unit)>;
    storage s;
    auto p = new (&s) Unit{1,2};
    std::cout << rp->id << "\n"; 
    p->~Unit(); 
    p = new (&s) Unit{2,2};
    std::cout << p->id << "\n";
    p->~Unit(); 
```

Но поддерживать указатель, взвращенный оператором `new`, не всегда возмножно. 
Он занимает место, его надо хранить, что неэффективно при реализации `optional`: для `int32_t` будет нужно в три раза больше места на 64хбитной системе (4 байта на `storage` + 8 байт на указатель)!

Поэтому в стандартной библиотеке с C++17 есть функция "отмывания" невесть откуда взявшихся указателей -- `std::launder`.

```C++
    using storage = std::aligned_storage_t<sizeof(Unit), alignof(Unit)>;
    storage s;
    new (&s) Unit{1,2};
    std::cout << std::launder(reinterpret_cast<Unit*>(&s))->id << "\n"; 
    std::launder(reinterpret_cast<Unit*>(&s))->~Unit(); 
    new (&s) Unit{2,2};
    std::cout << std::launder(reinterpret_cast<Unit*>(&s))->id << "\n"; 
    std::launder(reinterpret_cast<Unit*>(&s))->~Unit(); 
```

Так и при чем тут `const`? "Настоящая" константность (переменные и поля объявленные с `const`) вместе с UB при использовании "неправильных" указателей, как раз и позволяют компилятору производить описанные спецэффекты.


## Полезные ссылки
1. https://isocpp.org/wiki/faq/const-correctness
2. https://miyuki.github.io/2016/10/21/std-launder.html
3. https://stackoverflow.com/questions/123758/how-do-i-remove-code-duplication-between-similar-const-and-non-const-member-func