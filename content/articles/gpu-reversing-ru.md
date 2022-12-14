---
title: "Реверс-инжиниринг графического процессора 2009 года"
summary: "(powervr5)"
---


В связи с недавними усилиями, направленными на Asahi Linux, особенно на часть обратного проектирования графического процессора, у меня возникла идея попытаться посмотреть, смогу ли я попробовать что-то в этом духе на некотором оборудовании, которое у меня есть. После небольшого поиска в старых коробках я нашел несколько старых телефонов и несколько старых настольных графических процессоров, настольные в основном уже работают с драйверами r600 и Nouveau в mesa/linux, то же самое касается большинства мобильных устройств, которые имеют ассортимент графических процессоров Adreno и Mali, за одним исключением.

Мой Galaxy Tab 2.0 7.0 (P3110) основан на SoC Texas Instruments OMAP 4430, который включает в себя PowerVR SGX540 от Imagination Technologies, этот конкретный графический процессор относится к их поколению «Series5». Любой, кто немного знаком с миром драйверов с открытым исходным кодом, вероятно, знает, что Imagination хорошо известна тем, что выпускает только бинарные драйверы для своих графических процессоров (даже если они недавно начали объединять драйвер Vulkan в mesa). Удивительно, но часть ядра драйвера графического процессора имеет открытый исходный код, так что это будет весьма полезно.

Еще одна важная часть этого заключается в том, что произошла большая утечка исходного кода кода драйвера той же эпохи, в которой был выпущен этот графический процессор, но из-за сомнительной законности публикации полученной из него информации я решил использовать 100% чистый подход обратного проектирования.

# Начиная

Давайте посмотрим, на что способен GPU, после беглого поиска мы видим, что он поддерживает OpenGL ES 2.0, это означает, что он поддерживает шейдеры. Для тех, кто не знаком, грубый обзор того, как 3D-рендеринг обычно работает на компьютерах, заключается в том, что графический процессор получает список «примитивов», примитивы, как правило, представляют собой треугольники, квадраты или полосы любого из них, и превращает их в пиксели. Эта процедура была довольно жесткой еще во времена OpenGL 1.0, но когда возникла потребность в более сложных и реалистичных сценах, были введены «шейдеры», шейдеры, по сути, являются точками в конвейере преобразования примитивов -> пикселей, где программист может указать пользовательские код для изменения результатов рендеринга.

Чтобы быть более конкретным, при рендеринге с использованием OpenGL 2.0 программист имеет доступ к двум типам шейдеров, **вершинным шейдерам** и **фрагментным шейдерам**, шейдеры — это программы, которые запускаются один раз для каждого пикселя, покрытого примитивом (представьте, например, внутреннюю часть закрашенного треугольника). Вершинные шейдеры обычно используются для перемещения или вращения объектов или выполнения над ними деформации, в то время как фрагментные шейдеры обычно используются для освещения и текстурирования.

Это означает, что, как и у большинства других программируемых аппаратных графических процессоров, есть свой собственный язык ассемблера, и самое интересное заключается в том, что сборка графического процессора, как правило, намного более уникальна, чем код сборки процессора, в целом существует больше степеней свободы для проектирования, например. Графические процессоры Intel имеют высокоуровневые инструкции управления потоком (циклы, операторы if).

Итак, это ставит перед нами первую (и, возможно, самую важную) цель — узнать, как именно выглядит ассемблерный код, работающий на этом графическом процессоре.

Сначала давайте посмотрим, говорит ли производитель что-нибудь полезное об этом, если мы будем искать документацию, относящуюся к конкретному производителю, мы можем найти этот [pdf файл](https://web.archive.org/web/20190712215029/http://cdn.imgtec.com/sdk-documentation/PowerVR+Series5.Architecture+Guide+for+Developers.pdf) , который содержит очень общую блок-схему графического процессора и некоторую информацию о том, как в нем работает 3D-рендеринг, конечно, не достаточно, чтобы сделать что-нибудь полезное в данный момент. Но есть некоторая хорошая исходная информация, во-первых, теперь мы знаем, что архитектура шейдеров унифицирована , что означает, что вершинные и фрагментные шейдеры используют один и тот же ассемблерный код (это не так для многих старых графических процессоров), во-вторых, мы можем видеть что графический процессор запускает «микроядро», которое, по-видимому, обрабатывает прерывания.

Они также, кажется, предоставляют «инструмент профилирования шейдеров», который может показать разборку шейдеров на графических процессорах PowerVR, но своими словами:

![M](/img/powervr5-1/nda.png)

Что ж, информации было немного, но определенно больше, чем я ожидал от них.

# Приступаем к работе

Одна вещь, которая очень важна для этого процесса, заключается в том, что планшет в данный момент работает под управлением Android (он рутирован и работает с пользовательским ПЗУ), поэтому весь реверс-инжиниринг на данный момент должен быть выполнен с помощью Android, и часть тестовая программа должна быть написана на Java, в целом Android стал причиной многих проблем. Хорошо, что Google предоставляет очень простой пример OpenGL ES с использованием Android Native Development Kit и C++ здесь.

Первым шагом к обратному проектированию архитектуры является наличие для нее двоичных файлов, что не так просто, как кажется. В OpenGL основным способом компиляции шейдеров является компиляция во время выполнения, а затем их объединение в «программу». Мне немного повезло, так как OpenGL ES 2.0 имеет расширение «OES_get_program_binary», которое поддерживается реализацией драйвера на Android (скорее всего, предназначено для использования 3D-приложениями для кэширования шейдеров, поскольку мощность мобильных устройств довольно ограничена), поэтому получение Программные двоичные файлы так же просто, как скомпилировать их и вызвать «glGetProgramBinaryOES» с программой и другими соответствующими данными в качестве аргумента, код для этого:

```
    GLint length = 0;
    glGetProgramiv(program, GL_PROGRAM_BINARY_LENGTH_OES, &length);
    std::vector<GLubyte> buffer(length);
    GLenum format = 0;
    glGetProgramBinaryOES(program, length, NULL, &format, buffer.data());
    ```

...
Продолжение следует.
