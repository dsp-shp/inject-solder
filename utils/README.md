# **Модуль «utils/»**
*«Модуль, реализующий методы детекции, логирования и unit-тестирования»*

Модуль включает в себя следующие компоненты:
- ```detector.py``` – реализация детекции объекта
- ```logger.py``` – система логирования
- ```tester.py``` – система авто и unit–тестирования

Далее каждый из компонентов рассматривается детальнее.

# detector.py
*«Детекция фрагмента»*

Функция компонента использует SIFT детектор для определения ключевых точек и дескрипторов объекта и изображения. Дальнейший мэтчинг дескрипторов производится при помощи набора FLANN алгоритмов по принципу поиска k ближайших соседей (на текущий момент FLANN реализация является самой оптимальной, а k=2 – более чем достаточным). Очистка итоговых мэтчей осуществляется при помощи порогового отношения Lowe с коэффициентом 0.7, а проекция мэтчей – при помощи встроенного в OpenCV механизма гомографии. Углы и смещения объекта на изображении определятся планиметрически при помощи координат проекции, построенной на предыдущем шаге.

```python
from logger import logger

@logger ### декоратор системы логирования
def detector(
    query_path:str,
    train_path:str,
    params:dict=PARAMS, ### глобально предопределенный набор параметров
    logfile:bool=True,
    log:object=None,
    **kwargs ### экстра параметры
) -> dict:
    ...
```

Функция принимает следующие параметры:
- ```query_path:str``` – путь к эталонному изображению (объекту)
- ```train_path:str``` – путь к исследуемому изображению (изображению)
- ```params:dict``` – конфигурационный словарь с рядом тюнинг параметров:
  - ```MIN_MATCHES:int``` – минимально необходимое количество мэтчей для определения местоположения объекта на изображении и обработки исключений,
  - ```FLN_INDEX:dict``` – оптимизационные параметры индексации FLANN алгоритмов,
  - ```FLN_SEARCH:dict``` – оптимизационные параметры поиска FLANN алогритмов,
  - ```HOMOGRAPHY:dict``` - оптимизационные параметры определения гомографии;
- ```logfile:bool``` – способ логирования процесса выполнения:
  - ```= True``` - в файл директории ```logs/```
  - ```= False``` - в sys.stdout
- ```log:object``` – объект, реализующий логирование:<br>
*«По умолчанию ```log=None```, в случае использования своей системы логирования, необходима реализация на ее стороне методов ```info(...)``` и ```process(...)»```*

Функция возвращает словарь вида:
```python
{"x1":x1, "y1":y1, "x2":x2, "y2":y2, "r":r} ### где x1, y1, ... – некоторые float значения
```
... и выводит в stdout список (в дальнейшем он считывается внутри C# окружения), содержащий в себе значения смещения и поворота фрагмента относительно изображения, например:
```
[-0.1, -0.2, 3.4, 4.5, -15.0]
```
Где [0] и [1] элементы – координаты верхнего левого угла фрагмента, [2] и [3] – координаты правого нижнего угла фрагмента, и [4] элемент – его поворот с положительным значением для направления по часовой стрелке.

# logger.py
*«Система логирования»*

Система логирования реализуется как класс-декоратор поверх выполняемой задачи, и предоставляет следующие методы:
- ```init(...)``` – инициализация системы и определение параметров логирования
- ```info(...)``` – непосредственно логирование
- ```process(...)``` – обработка изображений
- ```transfer(...)``` – бэкапирование изображений в ```logs/``` директорию, перегружает ```process(...)``` функцию в случае с ```logfile=True```
- ```retain(...)``` – удаление неактуальных дневных партиций в ```logs/``` директорию

```python
class logger(object):
    ### Инициализация и метод вызова для декорации
    def __init__(self, func) -> None: self.func = func    
    def __call__(self, *args, **kwargs): return self.init(self.func, *args, **kwargs)
    ### Реализация системных методов
    def init(self, func:callable, *args, attempt:int=1, **kwargs) -> None:
        ...
    def info(self, message:str, *args, prefix:bool=True, status:str='INFO', divide:bool=False) -> None: 
        ... ### фактически реализован как self.lambda
    def process(self, path:str, type:str) -> str: 
        ...
    def transfer(self, path:str, type:str) -> str: 
        ... ### фактически реализован как self.lambda
    def retain(self, days:int=3) -> None:
        ...
```

Процесс жизненного цикла системы выглядит следующим образом:
1. Определение типа логирования: запись в лог-файл или вывод в stdout;
1. Определение даты выполнения и идентификатора (форматированный datetime) задачи;
1. Если выполняется логирование в файл ~> выделение в ```logs/``` директории соответствующей поддиректории, например, ```logs/2023-03-01/```, и создание лог-файла, например, ```logs/2023-03-01/230301-044839.log```;<br>Если выполняется логирование в stdout ~> переход к следующему шагу;
1. Вызов дочернего процесса (например, детекции) с передачей ему логирующего объекта – ```self``` экземпляра;
1. В случае возникновения ошибки в дочернем процессе ~> повторный (рекурсивный) вызов ```self.init(...)``` метода со скорректированными параметрами.

Так, в результате логирования (с вызовом ```detect()``` и ```logfile=True```) в соответствующей поддиректории можно увидеть сам лог-файл, бэкапы исходных изображений и результирующее мэтч-изображение с одним и тем же идентификатором задачи в названии.

# tester.py
*«Система авто и unit-тестирования»*

Система unit-тестирования реализуется как набор классов (наследующих ```unittest```) и методов для определения корректности выполнения в ряде разных случаев.

```python
### Тестирование детекции
class detectorTest(unittest.TestCase):
    def test_0(...) -> None: ...
    def test_1(...) -> None: ...
    def test_2(...) -> None: ...
    ...

### Тестирование методов логирования
class loggerTest(unittest.TestCase):
    def test_info(self) -> None: ...
    def test_process(self) -> None: ...
    def test_retain(self) -> None: ...
    ...
```

Проверка корректности работы компонента осуществляется вызовом соответствуюшего скрипта из терминала/командной оболочки:
```bash
python3 tester.py
```
<br>

Автотестирование реализуется при помощи методов класса ```auto_detector``` и набора подготовленных данных, помещаемых в ```/tests/autos```. 
```python
### Auto-тестирование детекции
class auto_detector:
    
    def __call__(self, ...) -> None: 
        data = ... ### получение данных при помощи generate() и compute()
        ### Сохранить результат
        pd.DataFrame(data).to_csv(path_or_buf='.../logs/tester.auto_detector.tsv', sep='\t', ...)

    ### Генерация возможных сценариев
    def generate(self, ...) -> tuple: 
        for files in types: ### для каждого типа (набор файлов с, без подсветки)
            for file in files: ### для каждого файла
                for scale in scales: ### для каждого случайного фрагмента
                    for angle in angles: ### для каждого угла поворота
                        for angle in angles: ### для каждой попытки
                            yield (...)

    ### Расчет координат, вызов детекции
    def compute(self, ...) -> list: ... return [...]
```

Тестирование вызывается скриптом:
```python
python3 -c "from inject_solder.utils import *; auto_detector()()"
```
И возвращает ```.tsv``` файл, сохраняемый в ```logs/``` директорию. Файл представляет из себя некоторый набор параметров, используемых при тестировании, предполагаемый и получаемый результаты детекции:

|img_name|img_size|img_rot_size|img_contains|img_is_highlighted|obj_scale|obj_size|obj_cords|obj_angle|obj_attem|pre_cords|det_angle|det_cords|det_query_keys|det_train_keys|det_matches|det_good_matches|
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
|image.jpg|665x512|775x668|TRUE|FALSE|25|166x128|(...)|-15|1|(...)|15.1|(...)|50|100|50|25|---|

Атрибуты файла имеют следующий смысл:
- ```img_name``` - наименование исходного изображения
- ```img_size``` - размер исходного изображения
- ```img_rot_size``` - размер изображения после поворота на некоторый угол
- ```img_contains``` - содержит ли изображение определяемый фрагмент
- ```img_is_highlighted``` - флаг подсветки исходного изображения (определяется по названию)
- ```obj_scale``` - масштаб фрагмента (выбирается произвольно в диапазоне от 25 до 50)
- ```obj_size``` - размер фрагмента
- ```obj_cords``` - координаты фрагмента на исходном изображении (без поворота последнего)
- ```obj_angle``` - угол поворота изображения
- ```obj_attem``` - случайное изображение (номер попытки)
- ```pre_cords``` - предполагаемый результат детекции (x1, y1, x2, y2)
- ```det_angle``` - получаемый детекцией угол
- ```det_cords``` - получаемый результат детекции (x1, y1, x2, y2)
- ```det_query_keys``` - количество ключевых точек фрагмента
- ```det_train_keys``` - количество ключевых точек изображения
- ```det_matches``` - количество мэтчей
- ```det_good_matches``` - количество мэтчей, соответствующих критерию Lowe