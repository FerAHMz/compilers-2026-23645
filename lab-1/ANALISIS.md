# 🧪 Análisis — Laboratorio 1: Introducción a ANTLR

En este documento presento mi análisis de la gramática `MiniLang.g4` y del archivo `Driver.py` del entorno base de ANTLR, explicando lo que entiendo sobre su funcionamiento.

---

## 1. ¿Qué es un archivo `.g4`?

Un archivo `.g4` es la **definición de una gramática** para ANTLR. A partir de él, ANTLR *genera automáticamente* el código del **lexer** (analizador léxico) y del **parser** (analizador sintáctico) en el lenguaje objetivo (aquí, Python 3).

### Secciones de un archivo `.g4`

1. **Declaración de la gramática** — la primera línea: `grammar MiniLang;`. El nombre debe **coincidir exactamente** con el nombre del archivo (`MiniLang.g4`); si no, ANTLR marca error.
2. **Reglas del parser (parser rules)** — empiezan con **minúscula** (`prog`, `stat`, `expr`). Describen la *estructura* del lenguaje (cómo se combinan los tokens).
3. **Reglas del lexer (lexer rules / tokens)** — empiezan con **MAYÚSCULA** (`ID`, `INT`, `NEWLINE`, `WS`). Describen los *símbolos básicos* (cómo se forman las palabras a partir de caracteres).

> Regla clave de ANTLR: **minúscula = parser, MAYÚSCULA = lexer.**

---

## 2. Análisis de `MiniLang.g4`

```antlr
grammar MiniLang;

prog:   stat+ ;

stat:   expr NEWLINE                 # printExpr
    |   ID '=' expr NEWLINE          # assign
    |   NEWLINE                      # blank
    ;

expr:   expr ('*'|'/') expr          # MulDiv
    |   expr ('+'|'-') expr          # AddSub
    |   INT                          # int
    |   ID                           # id
    |   '(' expr ')'                 # parens
    ;

MUL : '*' ;
DIV : '/' ;
ADD : '+' ;
SUB : '-' ;
ID  : [a-zA-Z]+ ;
INT : [0-9]+ ;
NEWLINE:'\r'? '\n' ;
WS  : [ \t]+ -> skip ;
```

### 2.1 Reglas del parser

- **`prog: stat+ ;`** — La **regla inicial** (start rule). Un programa es *una o más* sentencias (`stat`). El `+` significa "una o más repeticiones".
- **`stat`** — Una sentencia puede ser de tres formas:
  - `expr NEWLINE` → una expresión seguida de salto de línea (etiqueta `# printExpr`).
  - `ID '=' expr NEWLINE` → una asignación, p. ej. `a = 4` (etiqueta `# assign`).
  - `NEWLINE` → una línea en blanco (etiqueta `# blank`).
- **`expr`** — Una expresión, definida de forma **recursiva**:
  - `expr ('*'|'/') expr` → multiplicación o división.
  - `expr ('+'|'-') expr` → suma o resta.
  - `INT` → un número entero.
  - `ID` → un identificador (variable).
  - `'(' expr ')'` → una expresión entre paréntesis.

### 2.2 El uso de `#` (etiquetas de alternativas)

El símbolo `#` asigna una **etiqueta a cada alternativa** de una regla (p. ej. `# MulDiv`, `# AddSub`). Sirve para que ANTLR genere un **método separado** (`enterMulDiv`, `exitMulDiv`, `visitMulDiv`, …) por cada alternativa, en lugar de un único método para toda la regla.

> **"Utilizar `#` en ANTLR sirve para** etiquetar cada alternativa de una regla y así poder manejarla de forma independiente en el Listener o Visitor, en vez de tener que distinguir manualmente qué caso ocurrió."

Entiendo que sin las etiquetas tendría un solo `visitExpr` con toda la lógica mezclada; con ellas, cada operación tiene su propio método limpio.

### 2.3 Precedencia de operadores

ANTLR resuelve la **recursión por la izquierda** y aplica precedencia según el **orden** en que aparecen las alternativas: la que aparece **primero tiene mayor precedencia**. Como `('*'|'/')` está listada **antes** que `('+'|'-')`, la multiplicación y la división se evalúan antes que la suma y la resta.

Ejemplo: `3 + 4 * 2` se interpreta como `3 + (4 * 2)`.

### 2.4 Reglas del lexer (tokens)

- **`MUL`, `DIV`, `ADD`, `SUB`** — definen los tokens para `*`, `/`, `+`, `-`.
- **`ID : [a-zA-Z]+ ;`** — un identificador: una o más letras. `[a-zA-Z]` es una *clase de caracteres*.
- **`INT : [0-9]+ ;`** — un entero: uno o más dígitos.
- **`NEWLINE : '\r'? '\n' ;`** — un salto de línea. El `?` hace que `\r` (retorno de carro, típico en Windows) sea **opcional**; se usa como señal de *fin de sentencia*.
- **`WS : [ \t]+ -> skip ;`** — espacios y tabulaciones. La acción **`-> skip`** hace que el lexer los **descarte** para que no lleguen al parser (así los espacios no afectan el análisis).

---

## 3. Análisis de `Driver.py`

```python
import sys
from antlr4 import *
from MiniLangLexer import MiniLangLexer
from MiniLangParser import MiniLangParser

def main(argv):
    input_stream = FileStream(argv[1])          # 1. lee el archivo de entrada
    lexer = MiniLangLexer(input_stream)          # 2. lexer: caracteres -> tokens
    stream = CommonTokenStream(lexer)            # 3. buffer de tokens
    parser = MiniLangParser(stream)              # 4. parser: tokens -> árbol
    tree = parser.prog()                         # 5. invoca la regla inicial 'prog'

if __name__ == '__main__':
    main(sys.argv)
```

El `Driver.py` es el **punto de entrada** que conecta las piezas generadas por ANTLR:

1. **`FileStream(argv[1])`** — lee el archivo pasado como argumento en la terminal (p. ej. `program_test.txt`) y lo convierte en un flujo de caracteres.
2. **`MiniLangLexer(input_stream)`** — el lexer (generado por ANTLR desde la gramática) convierte los caracteres en **tokens** (`INT`, `ID`, `ADD`, `NEWLINE`, …).
3. **`CommonTokenStream(lexer)`** — almacena los tokens en un buffer que el parser puede recorrer.
4. **`MiniLangParser(stream)`** — el parser (también generado) recibe los tokens.
5. **`parser.prog()`** — invoca la **regla inicial** `prog`, que construye el **árbol de análisis sintáctico** (parse tree) verificando que la entrada cumpla la gramática.

> Nota: `MiniLangLexer` y `MiniLangParser` **no existen hasta que se ejecuta** `antlr -Dlanguage=Python3 MiniLang.g4`. Ese comando los genera a partir de la gramática.

### ¿Por qué a veces no muestra nada?

Si la entrada es **sintácticamente correcta**, el parser construye el árbol sin errores y el driver termina en silencio (no imprime el árbol). Si hay **errores de sintaxis**, ANTLR los reporta automáticamente en la consola indicando **línea y columna**.

---

## 4. Pruebas para observar el comportamiento

Para ver cómo responde el parser, probé 2 entradas válidas y 2 inválidas.

| Archivo | Resultado | Explicación |
|---|---|---|
| `good_1.txt` | ✅ Sin errores | Expresiones aritméticas válidas; respeta precedencia (`*`/`/` antes que `+`/`-`) y paréntesis. |
| `good_2.txt` | ✅ Sin errores | Asignaciones válidas `ID = expr` que reutilizan variables definidas. |
| `bad_1.txt` | ❌ Error | Operadores sin operando: `7 *` sin valor a la derecha y `3 + + 2` con dos operadores seguidos. |
| `bad_2.txt` | ❌ Error | `= 5` empieza con `=` sin identificador; `@` no es un token válido (token recognition error). |

**Errores reportados por ANTLR:**

```
bad_1.txt:
  line 1:3 extraneous input '\n' expecting {'(', ID, INT}
  line 2:4 extraneous input '+' expecting {'(', ID, INT}

bad_2.txt:
  line 1:0 extraneous input '=' expecting {'(', ID, INT, NEWLINE}
  line 2:6 token recognition error at: '@'
  line 2:8 extraneous input '2' expecting NEWLINE
```

---

## 5. Lo que entendí

De este material del curso entiendo que ANTLR separa claramente el **léxico** (tokens en MAYÚSCULA) de la **sintaxis** (reglas en minúscula) dentro de un único archivo `.g4`, y que a partir de esa gramática genera automáticamente el lexer y el parser. Comprendí que las etiquetas `#` sirven para manejar cada alternativa por separado (útil más adelante con Listeners/Visitors), que la precedencia de operadores se define por el **orden** de las alternativas, y que `-> skip` descarta el ruido léxico como los espacios. También entendí que el `Driver.py` es solo el punto de entrada que orquesta el flujo *archivo → lexer → tokens → parser → árbol*, y que es ANTLR quien reporta por sí mismo los errores de sintaxis indicando línea y columna.
