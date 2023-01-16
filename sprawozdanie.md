## Flaga
Wyszukałem funkcję `player_mailbox` i 2 razy kliknąłem na tekst pod
`aNothingHereMai`. IDA przechodzi do części kodu ze stałymi, pod którymi są
różne teksty/kwestie. W szczególności, parę linijek niżej, widnieje
`aFlagGr3ppNgThr`, pod którą jest flaga:
FLAG{gr3pp!ng-thr0ugh-str1ngs?-isn',27h,'t-th4t-t0o-ez?}.

## Wyświetlenie flagi w grze
W funkcji `player_mailbox` wywoływana jest funkcja `check` z argumentem 5,
a następnie, jeśli jej wynik jest 0, to wypisany zostanie tekst
`aNothingHereMai`. Żeby tekst ten nie został wypisany, `check` musi zwrócić
coś niezerowego. Widzimy, że `check` sprawdza, czy szósty od prawej bit jakiejś
zmiennej jest ustawiony na 1. Tę zmienną modyfikują funkcje `mark` oraz `clear`,
przy czym `mark` ma szansę zmienić bit z 0 na 1. Po przeszukaniu kodu można
znaleźć jedyne miejsce, w którym wykonane zostanie `mark(5)`, a konkretniej
w funkcji `overworld_keypress`. Stanie się to, jeśli pewien licznik osiągnie
wartość 14, a zwiększany jest po klikaniu odpowiednich klawiszy. Wciśnięty
klawisz jest binarnie xorowany ze stałą 106, a następnie wynik porównywany do
kolejnych elementów tablicy: `[9, 11, 4, 3, 2, 11, 16, 12, 6, 11, 13, 26, 6, 18]`.
Odwracając tę operację w celu znalezienia odpowiedniej kombinacji klawiszy
okaże się, że należy wpisać `canihazflagplx`. Następnie, skrzynka na listy
gracza zamiast komunikatu `aNothingHereMai` będzie pokazywać flagę.

## Chodzenie przez ściany
Aby przechodzić przez ściany bez użycia Shift, należy zmodyfikwoać funkcję
object_can_move tak, żeby zwracała zawsze `true`. Funkcja ta returnuje w dwóch
miejscach, w pierwszym robiąc wcześniej `mov al, 1`, natomiast w drugim
`xor al, al`. Wystarczy zmienić tę drugą instrukcję na `mov al, 1` i gotowe.

## Chodzenie przez ściany trzymając klawisz Shift (lewy)
Aby przechodzić przez ściany trzymając Shift, należy zmodyfikować funkcję
`player_step`. Wywoływana jest ona z funkcji `handle_movement_input`, która
czyta klawisz i wywołuje `player_step` z odpowiednim arguemntem. Idea
rozwiązania jest następująca:
1. Na początku `player_step` wrzuć na stos informację o naciśnięciu Shift.
2. Oprócz `call object_can_move` sprawdź informację o Shift na stosie, jeśli
klawisz jest wciśnięty, to wykonaj `mov al, 1`, aby móc przejść przez ścianę.
3. Na końcu funkcji trzeba poprawić obydwie instrukcje `add rsp,20` na
`add rsp,28`.

Rozwiązanie:
1. Podmieniamy `push rbx` na początku `player_step` na `jmp my_code_1`.
2. Podmieniamy `call object_can_move` na `jmp my_code_2`.
3. Podmieniamy `add rsp,20` na `add rsp,28` w obydwu miejscach.

Label `<my_code_1>`:
```
push rbx
push qword [rax+0xE1]
jmp  tutaj_1
```
Label `<my_code_2>`:
```
call object_can_move
cmp  byte [rsp+0x20],0
je   tutaj_2
mov  al,1
jmp  tutaj_2
```
Drobne uwagi:
- Label `<my_code_2>` jest w rzeczywistości rozbity na 3 labele, bo wszystkie
instrukcje razem nie zmieściły się w jednym bloczku `int3`.
- `tutaj_1` i `tutaj_2` to labele wskazujące odpowiednio na kolejną instrukcję
po `jmp my_code_1` i `jmp my_code_2` w funkcji `player_step`.
- Klawisz Shift jest pod `[rax+0xE1]`, co wiem po przeanalizowaniu funkcji
`handle_movement_input` oraz spojrzeniu tu ->
https://www.freepascal-meets-sdl.net/sdl-2-0-scancode-lookup-table/.
