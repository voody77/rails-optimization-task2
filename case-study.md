# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

После оптимизации по скорости она успешно обрабатывала файл в `130 MB` менее чем за `30 сек`, но при этом требовала около `3 GB` памяти для работы.

Я решил исправить эту проблему и продолжить оптимизировать эту программу с целью снизить объём потребляемой памяти. Конечно, существенная потеря производительности тоже недопустима.

## Метрика и Бюджет

Выбор метрики - это объем потребляемой памяти. Бюджет установим - `70 MB`

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста в фидбек-лупе позволяет не допустить изменения логики программы при оптимизации.

## Выстраиваем работу и Feedback-Loop

### Защита от регрессии производительности
Для защиты от регрессии производительности был написан `minitest` на проверку выделяемой памяти основным методом обработки `work`.
```ruby
class TestMe < Minitest::Test
  def memory_usage
    `ps -o rss= -p #{Process.pid}`.to_i / 1024
  end

  def test_result
    memory_usage_before= memory_usage
    work('tmp/data_80000.txt')
    memory_usage_after = memory_usage
    mem_usage = memory_usage_after - memory_usage_before
    puts "mem_usage: #{mem_usage} MB"
    assert_operator mem_usage, :<=, 90
  end
end
```
Для получения более точного значения этот тест лучше запускать отдельно, обернем в `Makefile`.

### Вывод memory usage

Для удобства добавим вывод информации о количестве используемой памяти при обработке основного файла:
```ruby
puts "MEMORY USAGE: #{`ps -o rss= -p #{Process.pid}`.to_i / 1024} MB"
```

### Пробуем профайлеры
При профилировании памяти важно **не отключать GC!!!**
Попробовал профилировать разными инструментами:

* `memory_profiler`
* `ruby-prof`
* `stackprof`
* `valgrind massif visualier`

На мой взгляд, `memory_profiler` - наиболее удобный инструмент для нашей задачи, который дает исчерпывающую информацию об аллокации памяти.
```ruby
# benchmark/memory_profiler.rb
report = MemoryProfiler.report do
  work('tmp/data_80000.txt', disable_gc: false)
end
report.pretty_print(scale_bytes: true)
```

Также удобно пользоваться отчетом `RubyProf::CallStackPrinter` в режиме `RubyProf.measure_mode = RubyProf::MEMORY`

```
# Makefile
memory_bench:
	ruby benchmark/memory_profiler.rb
	ruby benchmark/ruby-prof-memory.rb
```

Для того, чтобы убедиться, что программа на использует памяти больше заданного бюджета на протяжении всей работы будем использовать инструмент `valgrind massif visualier` (см `Dockerfile`, `build-docker.sh`, `profile.sh`, `visualize.sh`)
```
# Makefile
valgrind:
	./profile.sh && ./visualize.sh
```
![img/1.png](img/1.png)
**Рис 1.** Использование памяти в течение работы до оптимизации

### Находка №1
Из бюджета очевидно, что мы не можем ни считывать файл в память целиком, ни накапливать в памяти данные по пользователям.
Значит нам придется переписать программу в "потоковом" стиле - читаем исходный файл строку за строкой и сразу же на ходу пишем файл с результатами.

На данном этапе будет сложно переписать всю программу сразу, поэтому можно найти главные точки роста и оптимизировать их.

После анализа отчетов `memory_profiler` `ruby-prof` стало понятно, что основная точка роста памяти - это стоки, массивы и хеши.

Метод `String#split`, вызываемый для каждой строки создает массивы строк, которые в последствии не удаляются сборщиком мусора `GC` (что подтверждает **Рис 1.**).
Это из-за того, что мы складываем эту информацию в массив пользователей (в виде хешей). Т.е. настало время отказаться от этого массива пользователей и считать статистику по пользователям "на ходу". Это также позволит избавиться от хешей, которые создаются в методах `parse_user` и `parse_session`.

После рефакторинга удалось уменьшить объем потребляемой памяти с `2669 MB` до `1454 MB`. Также уменьшилось время обработки с `25s` до `16s`. Т.е. метрики уменьшились почти в 2а раза.
```
$ ruby work.rb # до
data_large Finish in 25.77
MEMORY USAGE: 2669 MB

$ ruby work.rb # после
data_large Finish in 15.62
MEMORY USAGE: 1454 MB
```
Обновляем `memory_test`

### Находка №2
Перепишем программу в "потоковом" стиле.
```
$ ruby work.rb
data_large Finish in 9.18
MEMORY USAGE: 15 MB
```
Метрики объем потребляемой памяти и время обработки уменьшились до `15 MB` и `10s` соответственно.

Обновляем `memory_test`

### Находка №3
Добавил `frozen_string_literal: true` - уменьшилось время обработки

```
$ ruby work.rb
data_large Finish in 8.74
MEMORY USAGE: 15 MB
```

## Результаты

В результате проделанной оптимизации удалось не только добиться цели и влезть в бюджет с запасом (`15 MB` против `70 MB`), но и улучшить время обработка файла с `25 секунд` до `10 секунд`.

![img/2.png](img/2.png)

Как видно из графика `valgrind massif visualier` - программа укладывается в бюджет на протяжении всего времени исполнения.