# Eksploitacja JitBF

Niniejszy artykuł dotyczy exploitacji programu jitbf który można znaleźć tutaj:
https://github.com/gynvael/stream/tree/03774e1f293f318c466a28b03735315c1a6c04b7/019-jitbf
Program powstał w trakcie streamu: https://www.youtube.com/watch?v=wJxWBeHWnGQ

## Jak doszedłem do ekspoita jako początkujący hakier
Na początku zauważyłem, że obszar JIT jest "rwx", czyli można tam zapisać kod a następnie wykonać. Założyłem, że jest to najlepsze pole do expoitacji. Pomijam takie rzeczy jak crash w przypadku błędnego kodu BF jak "[" bo to nie jest "wystarczająco fajne". Dosyć łatwo można zauważyć, że można w automacie BF wyjechać poza taśmę i nadpisać sobie wskaźnik ale tylko 1 bajtem. Niestety ale tutaj pomysł jest zły - kod który by przesunął się na początek JITa zajmuje za dużo miejsca: ciąg <<< lub >>> jest jitowany oraz zapisywany 2 bajty per znak oraz limitem 4kB.

Potem odkryłem, że przecież można nadpisać kod który jest kopiowany do JITa: globalne zmienne jit_ptr_inc oraz jit_ptr_dec z których jest kopiowany kod JIT nie są chronione. Oprócz tego te zmienne są bardzo blisko ctx (kilkadziesiąt bajtów oraz "po bliższej stronie"). Zmienne globalne są umieszczone całym blokiem pamięci zatem ich położenie względne jest niezależnie od adresu wirtualnego.

std::string w "starej" implementacji można opisać następująco:
```
struct {
  char *data;
  uint32_t size;
  union {
    uint32_t capacity;
    char buffer[16];
  }
};
```

Jeżeli data wskazuje na wewnętrzny bufor to logika kodu implementująca std::string zakłada tzw. krótki string oraz metody .data() oraz .size() zwracają w/w pola. Bardzo łatwo można nadpisać rozmiar stringa do 16 bajtów i nadpisać bufor własnym kodem - trzeba tylko uważać, aby nie używać atakowanego kodu. Więc zaatakowałem jit_ptr_inc poprzez sekwencję <.<. ... <<<. (nadpisuję kolejne bajty kodu z stdin shellcode'm oraz najmniej znaczące bajty rozmiaru do 0x0F). Tylko pytanie co się w takim małym obszarze zmieści - EB FE jako "jmp $" było pomocne w debugowaniu - zapisałem:
```
add esp,0x50 ; adres kodu BF na stosie: przesuwam się do aby na szczycie stosu był kod BF; znalezione w gdb
mov eax,0x7729b177 ; adres funkcji system wyciągnięty z gdb - zmienia się co reboot
call eax
```

Kod kompilowałem przy pomocy nasm. Ze swojej strony polecę https://www.onlinedisassembler.com/odaweb/ jako przenośny disasembler.

Następnie exploit uruchomiłem przez (^>^) z czego > jest najważniejszy, a na początku kodu BF umieściłem "calc.exe &:: " (to na końcu to początek komentarza dla cmd.exe).

Zauważyłem potem, że można uzyskać więcej miejsca poprzez nadużycie jit_ptr_inc: przecież następną strukturą w pamięci jest jit_ptr_dec który jest 2-bajtowym stringiem zatem w buforze ma 14 bajtów wolnego. Null na końcu można zignorować - jest używana metoda .size() do wyciągnięcia rozmiaru. Zatem nadpisałem rozmiar tak aby pokryć drugi bufor oraz zapisać kod do obu buforów. Trzeba mieć na uwadze to, że pomiędzy buforami są "święte dane" których nie wolno dotknąć czyli obsługa znaku "<" oraz trzeba w pierszym buforze zrobić skok do drugiego bufora. Miejsca nareszcie starczyło na wykonanie memcpy z kodu bf (a dokładniej kodu asm umieszczonego w pliku) do kodu JIT który miał prawa do zapisu w trakcie wykonania.

Zatem zapisałem kod:
```
mov edi,DWORD PTR [esp+0x14] ; Adres jit
mov esi,DWORD PTR [esp+0x50] ; Adres kodu BF
add esi,0x79 ; przesuń się z kodem BF na początek kodu asm zapisanego w pliku
nop ; filler
nop
nop
jmp 0x1a ; +0x1a aby ominąć dane
;[DANE] czyli ptr, size, 2 bajty kodu ważnego JITa i teraz drugi bufor
add edi,0x36 ; przesuń docelowy wskaźnik JIT na koniec "tego co będzie po nopach"
mov cx,0xff ; skopiuj 255 bajtów [można więcej tylko trzeba mieć na uwadze że jest 4kB miejsca minus kod użyty na JIT
rep movs ; memcpy
nop
nop
nop
nop
nop
```

To skopiowało kod z pliku wejściowego do kodu JIT zatem w tym miejscu mam "arbitrary code execution". No to dlaczego by nie uruchomić calc.exe? To z pomocą googla znalazłem shellcode który uruchamia calc.exe (oczywiście sprawdzić trzeba co to robi, nie wolno wykonywać dziwnego kodu na swojej maszynie).
