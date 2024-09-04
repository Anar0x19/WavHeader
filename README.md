Итак, рассмотрим самый обычный WAV файл (Windows PCM). Он представляет собой две, четко делящиеся, области. Одна из них — заголовок файла, другая — область данных. В заголовке файла хранится информация о:

Размере файла.
Количестве каналов.
Частоте дискретизации.
Количестве бит в сэмпле (эту величину ещё называют глубиной звучания).
Но для большего понимания смысла величин в заголовке следует ещё рассказать об области данных и оцифровке звука. Звук состоит из колебаний, которые при оцифровке приобретают ступенчатый вид. Этот вид обусловлен тем, что компьютер может воспроизводить в любой короткий промежуток времени звук определенной амплитуды (громкости) и этот короткий момент далеко не бесконечно короткий. Продолжительность этого промежутка и определяет частота дискретизации. Например, у нас файл с частотой дискретизации 44.1 kHz, это значит, что тот короткий промежуток времени равен 1/44100 секунды (следует из размерности величины Гц = 1/с). Современные звуковые карты поддерживают частоту дискретизации до 192 kHz. Так, со временем разобрались.

Амплитуда и сэмплы
Теперь, что касается амплитуды (громкости звука в коротком промежутке времени). Амплитуда выражается числом, которое занимает в файле 8, 16, 24, 32 бита (теоретически можно и больше). От точности амплитуды, я бы сказал, зависит точность звука. Как известно, 8 бит = 1 байту, следовательно, одно значение амплитуды в какой-то короткий промежуток времени в файле занимает 1, 2, 3, 4 байта соответственно. Таким образом, чем больше число занимает места в файле, тем шире возможный диапазон значений для этого числа, а значит и больше точность амплитуды.

Для PCM-файлов точность (или разрядность) может быть следующей:

1 байт / 8 бит — -128…127
2 байта / 16 бит — -32 760…32 760
3 байта / 24 бита — -1…1 (с плавающей точкой)
4 байта / 32 бита — -1…1 (с плавающей точкой)
Но список возможных разрядностей на самом деле весьма шире, здесь представлены лишь наиболее популярные.

Совокупность амплитуды и короткого промежутка времени носит название сэмпл.

Заголовок
Итак, давайте рассмотрим первую часть WAV-файла подробнее. Следующая таблица наглядно показывает структуру заголовка:

Местоположение	Поле	Описание
0…3 (4 байта)	chunkId	Содержит символы "RIFF" в ASCII кодировке 0x52494646. Является началом RIFF-цепочки.
4…7 (4 байта)	chunkSize	Это оставшийся размер цепочки, начиная с этой позиции. Иначе говоря, это размер файла минус 8, то есть, исключены поля chunkId и chunkSize.
8…11 (4 байта)	format	Содержит символы "WAVE" 0x57415645
12…15 (4 байта)	subchunk1Id	Содержит символы "fmt " 0x666d7420
16…19 (4 байта)	subchunk1Size	16 для формата PCM. Это оставшийся размер подцепочки, начиная с этой позиции.
20…21 (2 байта)	audioFormat	Аудио формат, список допустипых форматов. Для PCM = 1 (то есть, Линейное квантование). Значения, отличающиеся от 1, обозначают некоторый формат сжатия.
22…23 (2 байта)	numChannels	Количество каналов. Моно = 1, Стерео = 2 и т.д.
24…27 (4 байта)	sampleRate	Частота дискретизации. 8000 Гц, 44100 Гц и т.д.
28…31 (4 байта)	byteRate	Количество байт, переданных за секунду воспроизведения.
32…33 (2 байта)	blockAlign	Количество байт для одного сэмпла, включая все каналы.
34…35 (2 байта)	bitsPerSample	Количество бит в сэмпле. Так называемая "глубина" или точность звучания. 8 бит, 16 бит и т.д.
36…39 (4 байта)	subchunk2Id	Содержит символы "data" 0x64617461
40…43 (4 байта)	subchunk2Size	Количество байт в области данных.
44…	data	Непосредственно WAV-данные.
Вот и весь заголовок, длина которого составляет 44 байта.

Подводные камни
Выше мы рассмотрели простейший случай заголовка с одной подцепочкой перед областью данных. Но на практике встречаются и более сложные или даже непредвиденные сценарии, с которыми можно увязнуть надолго.

В chunkSize лежит заведомо слишком большое значение. Такое происходит, когда вы пытаетесь читать данные в режиме стриминга. Например, декодер LAME при выводе результата декодирования в STDOUT в этом поле возвращает значение 0x7FFFFFFF + 44 - 8, а в subchunk2Size — 0x7FFFFFFF (что равно максимальному значению 32-разрядного знакового целочисленного значения). Это объясняется тем, что декодер в таком режиме выдаёт результат не целиком, а небольшими наборами данных и не может заранее определить итоговый размер данных.

Подцепочек может быть больше, чем две, например, при попытке декодировать аудио универсальным декодером ffmpeg 4.1.3 ffmpeg -i example.mp3 -f wav example.wav в декодированном файле помимо рассмотренных подцепочек fmt и data будет содержаться ещё одна LIST перед областью данных. Таким образом, когда вам понадобится добраться до данных, вам потребуется пропустить ненужные подцепочки, пока не встретится data. Это будет сделать не слишком сложно, так как можно читать ID подцепочки и её размер, и если она не data, то пропускать данные, основываясь на её размере.

Блок данных
В моно варианте значения амплитуды расположены последовательно. В стерео же, например, сначала идет значение амплитуды для левого канала, затем для правого, затем снова для левого и так далее.

Заметка о типах данных
При чтении заголовка можно применять разные типы данных. Например, в Си (MSVS) вместо массива char[4] можно использовать __int32 или DWORD, но тогда сравнение с какой-либо строковой константой, к примеру может оказаться не очень удобным. Также хотелось бы предостеречь вас на тему 64-битных операционных систем. А именно: всегда стоит помнить, что в языке Си тип переменной int в 64-битной системе будет иметь длину 8 байт, а в 32-битной — 4 байта. В таких случаях можно воспользоваться вышеупомянутым типом переменной __int32 или __int64, в зависимости от того, какой размер переменной в памяти Вам необходим. Существуют типы __int8, __int16, __int32 и __int64, они доступны только для MSVC++ компилятора как минимум 7-й версии (Microsoft Visual Studio 2003.NET), но зато Вы не ошибетесь с выбором размера типа данных.
