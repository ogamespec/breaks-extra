# Сшивание с Fiji

Допустим нам нужно сшит такой датасет:

![dataset](/Others/imgstore/dataset.jpg)

Скачать 64-битную версию: https://imagej.net/software/fiji/

![fiji](/Others/imgstore/fiji.jpg)

Plugins -> Stitching -> Grid/Collection stitching

Выбрать порядок слайдов. Обычно слайды идут паттерном "змейка":

|Змейка сверху-вниз|Змейка снизу-вверх|
|---|---|
|![order1](/Others/imgstore/order1.jpg)|![order2](/Others/imgstore/order2.jpg)|

Теперь нужно установить:
- Количество слайдов по x (Grid size x)
- Количество рядов (Grid size y)
- Перекрытие слайдов (Tile overlap): подбираются опытным путем 5-20%
- First file index i: Если название слайды начинаются не с 1
- Directory: папка со слайдами
- File names for tiles: Паттерн для названия слайдов. В нашем случае: `{iiii}.jpg` (Имена слайдов начинаются с 0001.jpg и далее)
- Дальше ещё какие-то непонятные опции

![options](/Others/imgstore/options.jpg)

После нажатия на `Run` программа начинает колдовать:

![magic](/Others/imgstore/magic.jpg)

И выдаст картинку:

![fused](/Others/imgstore/fused.jpg)

Сохранить полученную картинку можно через File -> Save As.
