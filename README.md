# gtk — bindingi GTK4 dla H#

Bindingi do **GTK4** dla języka **H#** (H-Sharp). W **100% kod H#** — ani
jednej linii C. Cała biblioteka to deklaracje `extern static [c, "..."]`
wskazujące bezpośrednio na prawdziwe, systemowe `libgtk-4`, `libglib-2.0`
i `libgobject-2.0`, plus wygodna warstwa wysokopoziomowa (struktury,
`impl`, obsługa zdarzeń) napisana czystym H# na wierzchu tych deklaracji.

```hsharp
use "bit -> gtk" from "gtk"

fn main() is
    gtk::init()
    let win: gtk::Window = gtk::new_window("Witaj, H#!", 400, 200)
    let btn: gtk::Button = gtk::new_button("Kliknij mnie")
    win.set_child(btn.handle)

    let app: gtk::App = gtk::new_app()
    app.on_click(btn, || is write("Klik!") end)

    win.present()
    app.run()
end
```

## Instalacja

Wymagane w systemie (dowolna dystrybucja Linuksa z GTK4):

```bash
# Debian/Ubuntu
sudo apt install libgtk-4-dev libglib2.0-dev pkg-config

# Fedora
sudo dnf install gtk4-devel glib2-devel pkgconf-pkg-config

# Arch
sudo pacman -S gtk4 glib2 pkgconf
```

Następnie w swoim projekcie H#:

```bash
bit add gtk
```

lub ręcznie, dopisując do `[dependencies]` w swoim `Bit.hk`:

```
[dependencies]
-> gtk => newest
```

## Budowanie tej biblioteki samodzielnie

```bash
git clone <adres-repo> gtk
cd gtk
h# check src/gtk.h#              # weryfikacja składni/typów
h# compile examples/hello_window.h# --emit bin -o hello_window
./hello_window
```

Kompilator H# (`ffi_linker.rs`) dla każdego bloku `extern static [c, "<lib>"]`
najpierw próbuje `pkg-config --static --libs <lib>`; jeśli dana biblioteka
nie ma statycznego wariantu w systemie (typowe dla GTK4 na większości
dystrybucji — pakiety `-dev`/`-devel` zwykle dają tylko `.so`), linker i tak
poprawnie dowiąże się dynamicznie do `.so`, bo `pkg-config` bez `--static`
i tak zwraca poprawne `-lgtk-4 -lglib-2.0 -lgobject-2.0`. Blok jest
zadeklarowany jako `static`, zgodnie z życzeniem — jeśli w Twoim systemie
są dostępne statyczne `.a` dla GTK4/GLib, `bit`/`h#` użyje ich automatycznie.

## Struktura

Od 0.6 biblioteka jest rozbita na małe pliki (żaden nie przekracza ~400 linii):

```
gtk/
  Bit.hk                  manifest pakietu
  README.md, LICENSE
  src/
    gtk.h#                plik wejściowy (`lib-entry`): lista modułów + importy std
    ffi/                  surowe `extern static [c, "..."]` (dawny `gtk_ffi.h#`, 760 linii)
      mod.h#              lista modułów FFI
      ffi_core.h#         init, pętla GLib, GObject (ref, sygnały), GQueue, GIO, czas
      ffi_window.h#       GtkWindow, HeaderBar/ActionBar, AlertDialog, FileDialog
      ffi_widget.h#       GtkWidget (operacje wspólne)
      ffi_controls.h#     przyciski, etykiety, pola tekstowe, Check/Switch/Scale/Progress/Image
      ffi_containers.h#   Box, Grid, Stack, Notebook, Paned, Revealer, FlowBox, ...
      ffi_text.h#         TextView / TextBuffer
      ffi_lists.h#        DropDown, ListBox, GListModel
      ffi_misc.h#         MenuButton/Popover, CSS, Calendar, schowek
      ffi_adw.h#          libadwaita
      ffi_ext.h#          (0.6) dodatki
    <moduł>/              warstwa wysokopoziomowa; w każdym katalogu `mod.h#` + `gtk.h#`
      enums core window layout buttons text values lists nav
      app app_events app_watch app_async app_run          (struktura i metody `App`)
      adw adw_ext                                          (libadwaita)
      extras widgets_ext adw_rows app_ext                  (0.6)
  examples/
    hello_window.h#  counter_and_form.h#  widgets_showcase.h#  dashboard.h#
    dialogs_and_rows.h#   (0.6) confirm, schowek, SearchBar, SwitchRow/ComboRow/SpinRow, Avatar
```

### Dlaczego każdy moduł leży w katalogu jako `gtk.h#`?

Kompilator H# dokleja do nazw z pliku ładowanego przez `mod X` prefiks `X_`. Gdyby moduł
nazywał się `window`, funkcja `new_window` stałaby się `window::new_window` i **cała publiczna
składnia (`gtk::new_window`, `gtk::Window`, `app.on_click(...)`) by się zmieniła**. Dlatego
każdy plik warstwy wysokopoziomowej jest ładowany jako moduł o nazwie `gtk` (z osobnego
katalogu, bo nazwa pliku = nazwa modułu) i dostaje taki sam prefiks `gtk_` jak plik wejściowy —
API pozostaje płaskie i identyczne jak przed podziałem. Pliki FFI (`extern`) nie są
prefiksowane, więc ich nazwy modułów są dowolne.

Zasady pisania kodu w modułach:

- odwołania do funkcji z **innego** pliku: `gtk::nazwa(...)`, do typów: `gtk::Typ`;
- dodatkowy `impl` dla typu zdefiniowanego w innym pliku: `impl gtk_Typ is ... end`;
- `fn_addr(gtk::cb_file_done)` — callbacki C z innego pliku też kwalifikujemy;
- wywołania FFI zawsze jako `gtk_ffi::nazwa(...)`;
- **import biblioteki musi mieć alias `gtk`**: `use "bit -> gtk" from "gtk"` (z innym aliasem
  nazwy z plików modułów nadal zaczynałyby się od `gtk_`).

## Pokryte widgety

| Kategoria       | Widgety |
|-----------------|---------|
| Okna/kontenery  | `Window`, `Box`, `Grid`, `Frame`, `ScrolledWindow`, `CenterBox`, `Overlay`, `Paned`, `Revealer`, `Expander`, `FlowBox`, `HeaderBar`, `ActionBar` |
| Podstawowe      | `Button`, `ToggleButton`, `LinkButton`, `Label`, `Entry`, `PasswordEntry`, `SearchEntry`, `CheckButton`, `Switch`, `Image`, `Picture`, `Separator`, `TextView` |
| Wartości        | `ProgressBar`, `LevelBar`, `Spinner`, `Scale`, `SpinButton` |
| Wybór           | `DropDown`, `ListBox`, `Calendar` |
| Nawigacja       | `Stack` + `StackSwitcher`, `Notebook`, `MenuButton` + `Popover` |
| Dialogi         | `show_alert` (`GtkAlertDialog`, fire-and-forget) |
| Inne            | `clipboard_set_text`, `load_css`, timery `App::every` / `after` |

### Nowości w 0.6

- **podział na małe pliki** (patrz „Struktura”); API bez zmian — wygenerowany kod C++ dla
  starych przykładów jest identyczny z wersją sprzed podziału;
- `app.confirm(okno, wiadomość, szczegóły, "Anuluj|Usuń", |i: int| is ... end)` — okno z własnymi
  przyciskami i odczytem wyboru (`-1` = zamknięto), asynchroniczne (`GtkAlertDialog`);
- `app.read_clipboard(|tekst: string| is ... end)` — asynchroniczny odczyt schowka;
- `ScrolledWindow::scroll_to_top/scroll_to_bottom/scroll_value/set_scroll_value` (przez `GtkAdjustment`);
- `CheckButton::set_label/label/set_group/set_inconsistent` (radio-buttony);
- `TextView::set_padding/set_line_spacing/undo/redo/can_undo/can_redo/set_undo_enabled/insert_at_cursor`;
- `Grid::attach_next_to/insert_row/insert_column/remove_row/remove_column`, `Notebook::set_show_tabs`,
  `Paned::set_resize_*/set_shrink_*`, `Stack::set_homogeneous/set_interpolate_size`,
  `Revealer::child_revealed`, `DropDown::clear/set_items`, `ListBox::set_show_separators/row_at/select_all`,
  `Label::set_use_markup/set_single_line_mode/set_yalign`, `Entry::set_activates_default`,
  `Window::set_default_size/set_focus`, `ProgressBar::set_inverted`, `LevelBar::set_range`;
- wspólne: `gtk::set_halign/set_valign/set_hexpand/set_vexpand/set_tooltip_markup/set_overflow_hidden/activate/is_ancestor/scale_factor`;
- nowe widgety: `SearchBar`, `Fixed`, `WindowHandle`; libadwaita: `SwitchRow`, `ComboRow`, `SpinRow`, `Avatar`
  (+ `app.on_switch_row`, `app.on_combo_row`);
- wszystkie symbole FFI sprawdzone `nm -D` w GTK 4.14 / libadwaita 1.5; `Bit.hk` deklaruje teraz
  też zależność od libadwaita.

### Nowości w 0.2

- nowe widgety z tabeli wyżej; nowe metody m.in. `Window::close/set_titlebar/is_active`,
  `Box::clear/prepend`, `ListBox::select_index/clear`, `DropDown::selected_text/count`,
  `TextView::text/set_text/append`, `Label::set_markup`, wspólne `gtk::set_tooltip`,
  `set_opacity`, `set_cursor`, `widget_width/height`;
- nowe zdarzenia `App`: `on_switch`, `on_scale_change`, `on_page_change`, `on_toggle_button`,
  `on_text_change`, `on_date_change`, `on_expand`, `on_reveal`, `on_paned_move`,
  `on_focus`, `on_press`, `on_resize`, `on_window_active`;
- timery: `app.every(ms, cb)`, `app.after(ms, cb)`, `app.cancel(id)`;
- `app.unwatch(handle)` — wywołaj przed zniszczeniem obserwowanego widgetu;
- poprawka: `Frame::set_child` wołał niezadeklarowane `gtk_frame_set_child`;
- poprawka: wyniki `gint` (np. `-1`) są teraz poprawnie rozszerzane o znak;
- `use "bytes -> gtk"` → `use "bit -> gtk"` (parser H# odrzuca `bytes`).

Każdy widget to struktura H# z polem `handle: int` (surowy wskaźnik GTK —
tak samo jak `malloc`/`free` w rdzeniu H# reprezentują pamięć jako `int`)
oraz metodami `impl` wołającymi prawdziwe funkcje `gtk_*`/`g_*` z
`src/ffi/`. Wspólne dla każdego widgetu: `gtk::show`, `gtk::hide`,
`gtk::set_visible`, `gtk::set_sensitive`, `gtk::set_size`, `gtk::set_margin`,
`gtk::set_align`, `gtk::set_expand` (hexpand/vexpand), `gtk::add_css_class` /
`gtk::remove_css_class`, `gtk::grab_focus`.

## Style CSS

```hsharp
gtk::load_css("
    .naglowek { font-weight: bold; color: #3b82f6; }
")
gtk::add_css_class(label.handle, "naglowek")
```

albo z pliku: `gtk::load_css_file("styl.css")`. Obie funkcje owijają
prawdziwe `GtkCssProvider` + `gtk_style_context_add_provider_for_display`
z priorytetem `GTK_STYLE_PROVIDER_PRIORITY_APPLICATION`.

## Jak działają zdarzenia (bez ani jednej linii C)

GTK normalnie łączy interakcje użytkownika (klik, zmianę checkboksa, ...)
przez sygnały GObject (`g_signal_connect`), które wymagają C-owego
wskaźnika funkcji jako callbacku. Dzisiejszy kompilator H# nie generuje
takiej trampoliny z domknięcia H# (`ffi.rs::type_to_c` mapuje typ funkcyjny
tylko na nieprzezroczysty `void*`, bez żadnego wywołania zwrotnego C→H#).
Ponieważ ta biblioteka ma być **w 100% H#**, `App` w `gtk.h#` zamiast
sygnałów odpytuje (polling) prawdziwy stan widgetu w swojej pętli głównej:

| Metoda `App`          | Widget         | Co odpytuje                                        | Wiarygodność |
|------------------------|----------------|-----------------------------------------------------|--------------|
| `on_click`             | `Button`       | flaga `GTK_STATE_FLAG_ACTIVE` (zbocze)               | przybliżona* |
| `on_toggle`            | `CheckButton`  | `gtk_check_button_get_active`                        | 100% pewna   |
| `on_change`            | `Entry`        | `gtk_editable_get_text`                              | 100% pewna   |
| `on_hover`             | dowolny widget | flaga `GTK_STATE_FLAG_PRELIGHT`                      | 100% pewna   |
| `on_submit`            | `Entry`        | flaga `GTK_STATE_FLAG_FOCUS_WITHIN` (zbocze, utrata fokusu) | przybliżona** |
| `on_selection_change`  | `ListBox`      | `gtk_list_box_get_selected_row` + `row_get_index`    | 100% pewna   |
| `on_dropdown_change`   | `DropDown`     | `gtk_drop_down_get_selected`                         | 100% pewna   |
| `on_value_change`      | `SpinButton`   | `gtk_spin_button_get_value`                          | 100% pewna   |
| `on_view_change`       | `Stack`        | `gtk_stack_get_visible_child_name`                   | 100% pewna   |
| `on_window_close`      | `Window`       | `gtk_widget_get_mapped` (po `g_object_ref`, zbocze)  | przybliżona*** |

\* `App::run()` odpytuje w **każdej** iteracji prawdziwej pętli GLib
(`g_main_context_iteration`), więc w praktyce nie da się kliknąć szybciej
niż biblioteka to zauważy — ale formalnie jest to detekcja zbocza stanu, a
nie natywny sygnał `clicked`.

\*\* To NIE jest sygnał `activate` (Enter) — ten wymaga
`GtkEventControllerKey` + sygnału, czyli C-owego callbacku. `on_submit`
odpala się przy **utracie fokusu** (użytkownik kliknął gdzie indziej /
wcisnął Tab), co w praktyce często pokrywa się z "skończyłem wpisywać",
ale nie jest identyczne z wciśnięciem Enter.

\*\*\* Realny sygnał `close-request` wymaga C-owego callbacku. Zamiast
tego trzymamy dodatkową referencję (`g_object_ref`) na oknie, żeby było
bezpiecznie odpytywać jego stan `mapped` nawet po tym, jak domyślny
handler GTK zniszczy widget po kliknięciu w „X”. To działa poprawnie w
typowym przypadku (okno bez własnego `close-request`), ale jeśli chcesz
100% gwarancji, najprościej dodać własny przycisk "Zamknij"/"Wyjdź"
wołający `app.quit()` (patrz `examples/hello_window.h#`).

## Znane ograniczenia

- **Zdarzenia są głównie odpytywane (polling).** Od 0.4 `on_click` i `on_submit` używają prawdziwych
  sygnałów GObject (callbacki C to zwykłe funkcje H#, przekazywane przez `fn_addr`), a okna wyboru pliku,
  `confirm` i `read_clipboard` — asynchronicznych `GAsyncReadyCallback`. Pozostałe zdarzenia nadal
  porównują stan widgetów w każdej iteracji `run()`.
- **`gtk_alert_dialog_new` jest wariadyczne** — biblioteka przekazuje wiadomość jako format z
  zamienionym `%` na `%%`.
- **Wyciek przy odczycie `TextView`.** `gtk_text_buffer_get_text` zwraca świeżo zaalokowany `char*`;
  każde `TextView::text()` i `on_text_change` gubi kopię tekstu. To samo dotyczy tekstu z `read_clipboard`.
- **Timery** mają rozdzielczość ok. 4 ms; przy aktywnym timerze pętla nie blokuje się.
- **Brak `g_object_set`** (funkcja wariadyczna). Ciemny motyw: `gtk::prefer_dark`, `GTK_THEME=Adwaita:dark` albo CSS.
- **Domknięcia przekazywane do `on_*` nie mogą kończyć się samotnym `if` bez `else`** (błąd generatora
  C++ w kompilatorze H# 0.9) — dodaj na końcu dowolną instrukcję, np. `let _done: int = 0`.
- **Weryfikacja:** biblioteka i przykłady kompilują się do plików obiektowych (g++) oraz przechodzą
  `bit check`; nie była uruchamiana z wyświetlaczem — testy GUI wymagają GTK4 w systemie.

## Licencja

MIT — patrz `LICENSE`.
