### GUI-проиложение, написанное на Go с использованием GTK

Для написания десктопных приложений на языке Go существует несколько библиотек-привязок к графическим тулкитам, таким как Gtk и Qt. Есть еще пара-тройка написанных собственно на Go GUI-библиотек, однако они пока еще находятся на ранней стадии развития.

Сегодня я опробую библиотеку go-gtk (исходный код, документация), которая позволяет писать GTK+ 2 приложения. Прежде, чем устанавливать ее, убедитесь, что установлена переменная окружения GOPATH. Если нет, установите ее (например так: export GOPATH="$HOME/go"). Также вам, конечно, понадобится сама библиотека Gtk2. Если все готово для установки go-gtk, выполните следующую команду в консоли:

go get github.com/mattn/go-gtk/gtk
В качестве примера напишем каркас программы для заметок. Создайте пустой каталог и файл main.go в нем со следующим содержимым:

```golang
package main

import (
  "os"
  "github.com/mattn/go-gtk/gtk"
)

func main() {
  gtk.Init(&os.Args)
  gtk.Main()
}
```

Это минимальное Gtk приложение на Go. Здесь функция Init производит инициализацию тулкита, а Main запускает основной цикл, в котором Gtk ожидает и обрабатывает события по мере поступления.

В таком виде программа скомпилируется и запустится, но так и будет пребывать в состоянии ожидания пока ее не убьет пользователь. Чтобы наполнить ее существование смыслом добавим несколько виджетов и обработчиков событий. Но сначала для упрощения кода введем глобальную переменную, которая будет хранить номер последней вкладки:

var last int
Вставьте эту строку сразу после блока импорта.

Далее мы добавляем основное окно, задаем его заголовок и иконку, а также функцию выхода из программы в случае закрытия основного окна. Весь дальнейший код функции main будет расположен между gtk.Init(&os.Args) и gtk.Main().

```golang
window := gtk.NewWindow(gtk.WINDOW_TOPLEVEL)
window.SetTitle("Notes")
window.SetIconName("gtk-about")
window.Connect("destroy", func() {
  gtk.MainQuit()
})
```

Теперь создадим виджет notebook, добавим возможность прокрутки вкладок и создадим вкладку с иконкой в виде плюса, которая будет работать привычным всем образом, а именно создавать новую вкладку. Для этого мы проверяем, является ли нажатая вкладка последней, и если да, вызываем функцию добавления новой вкладки addPage, которая будет описана ниже.

```golang
notebook := gtk.NewNotebook()
notebook.SetScrollable(true)

notebook.AppendPage(gtk.NewVBox(false, 1),
  gtk.NewImageFromStock(gtk.STOCK_ADD, gtk.ICON_SIZE_MENU))

notebook.Connect("button-release-event", func() bool {
  n := notebook.GetCurrentPage()
  if n == last {
    addPage(notebook)
  }
  return false
})
```

В следующих строках виджет notebook добавляется в основное окно, задаются его минимальные размеры, окно со всеми виджетами выводится на экран.

```golang
window.Add(notebook)
window.SetSizeRequest(500, 300)
window.ShowAll()
```

Теперь рассмотрим функцию добавления новой вкладки/страницы addPage.

```golang
func addPage(notebook *gtk.Notebook) {
  dialog := gtk.NewDialog()
  dialog.SetTitle("Title?")
  dVbox := dialog.GetVBox()

  input := gtk.NewEntry()
  input.SetEditable(true)
  dVbox.Add(input)
  vbox := gtk.NewVBox(false, 1)
  ```
  
Первым делом запрашиваем у пользователя заголовок новой вкладки. Для этого создаем диалог, получаем вертикальный контейнер диалога и добавляем туда текстовое поле ввода input. Создаем еще один вертикальный контейнер для содержимого новой вкладки.

```golang
input.Connect("activate", func() {
  s := input.GetText()
  if s != "" {
      notebook.InsertPage(vbox, gtk.NewLabel(s), last)
      last++
      notebook.ShowAll()
  }
  notebook.PrevPage()
  dialog.Destroy()
})
```

Чтобы отловить нажатие кнопки Enter по окончании ввода, подключаем обработчик события «activate». Функция обработки проверяет, не пустая ли строка, иначе вкладка получится слишком узкой, вставляет новую вкладку и увеличивает счетчик. Вызов метода ShowAll отобразит все изменения. Вызов PrevPage сделает активной новую вкладку, после чего диалог уничтожается.

```golang
button := gtk.NewButtonWithLabel("OK")
button.Connect("clicked", func() {
  input.Emit("activate")
})
dVbox.Add(button)
dialog.SetModal(true)
dialog.ShowAll()
```

Здесь мы добавляем в диалог кнопку «ОК», а поскольку она будет дублировать нажатие Enter в строке ввода, вместо непосредственной обработки вызываем это событие программно. Дабы заблокировать основное окно на время ввода заголовка, сделаем диалог модальным.

Модальный диалог на Go

Так он будет выглядеть в действии.

```golang
hbox := gtk.NewHBox(false, 1)
swin := gtk.NewScrolledWindow(nil, nil)
swin.SetPolicy(gtk.POLICY_AUTOMATIC, gtk.POLICY_AUTOMATIC)
textview := gtk.NewTextView()
swin.Add(textview)
```

Этот код создает горизонтальный контейнер для кнопок, прокручиваемое окно и многострочный текстовый виджет.

```golang
buf := textview.GetBuffer()
tagBold := buf.CreateTag("bold", map[string]string{"weight": "700"})
tagItalic := buf.CreateTag("italic", map[string]string{"style": "2"})
Здесь мы получаем текстовый буфер и создаем пару тегов для изменения стиля текста.

butBold := gtk.NewToolButtonFromStock(gtk.STOCK_BOLD)
butBold.Connect("clicked", func() {
  var iter1, iter2 gtk.TextIter
  if buf.GetSelectionBounds(&iter1, &iter2) {
    buf.ApplyTag(tagBold, &iter1, &iter2)
  }
})

butIta := gtk.NewToolButtonFromStock(gtk.STOCK_ITALIC)
butIta.Connect("clicked", func() {
  var iter1, iter2 gtk.TextIter
  if buf.GetSelectionBounds(&iter1, &iter2) {
    buf.ApplyTag(tagItalic, &iter1, &iter2)
  }
})
  
butFont := gtk.NewFontButton()
butFont.Connect("font-set", func() {
  textview.ModifyFontEasy(butFont.GetFontName())
})
  
butClose := gtk.NewToolButtonFromStock(gtk.STOCK_DELETE)
butClose.Connect("clicked", func() {
  n := notebook.GetCurrentPage()
  notebook.RemovePage(notebook, n)
  last--
  notebook.PrevPage()
})
```

Еще пара кнопок из стандартного набора Gtk, кнопка смены шрифта для textview и закрытия вкладки с функциями обработки соответствующих сигналов. К сожалению, кнопки butBold и butIta мне не удалось заставить работать. Вместо изменения свойств выделенного текста они устанавливали их в дефолтное состояние.

```golang
  hbox.PackStart(butBold, false, false, 0)
  hbox.PackStart(butIta, false, false, 0) 
  hbox.PackStart(butFont, false, false, 0)
  hbox.PackEnd(butClose, false, false, 0)
  vbox.PackStart(hbox, false, false, 0)
  vbox.Add(swin)
  notebook.ShowAll()
}
```

Методы PackStart и PackEnd позволяют добиться большей гибкости в установке контролов в контейнерах. Второй аргумент отвечает за то, будет ли под данный виджет отведено все свободное пространство, третий — будет ли он его заполнять, последний аргумент позволяет увеличить интервал между контролами помимо заданного контейнером.

Вот и все, выполните go build в каталоге проекта и запустите полученный бинарник. После создания нескольких вкладок результат будет выглядеть примерно так:

GUI-проиложение, написанное на Go с использованием GTK

Код примера одним куском можно найти здесь (зеркало).

В целом, по моим впечатлениям, данная библиотека вполне пригодна для написания GUI-приложений на Go при условии, что вам хватит реализованных авторами 45% функциональности Gtk+ 2.

