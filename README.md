# Laboratorio 1: Introducción a ANTLR

**Curso:** Compiladores 2026
**Autora:** Anggie Quezada
**Modalidad:** Individual

## Descripción

Este laboratorio implementa y prueba una gramática ANTLR llamada **MiniLang**, que reconoce expresiones aritméticas simples y asignaciones de variables. El entorno se ejecuta dentro de un contenedor Docker que ya incluye Java, Python y ANTLR instalados.

## Estructura del repositorio

```
.
├── Dockerfile
├── program/
│   ├── MiniLang.g4              # Gramática ANTLR
│   ├── Driver.py                # Punto de entrada del análisis
│   ├── program_test_correct.txt # Caso de prueba válido
│   └── program_test_error.txt   # Caso de prueba con error sintáctico
└── README.md
```

## Cómo ejecutar el laboratorio

### 1. Construir y levantar el contenedor

```bash
docker build --rm . -t lab1-image && docker run --rm -ti -v "$(pwd)/program":/program lab1-image
```

### 2. Generar el lexer y el parser

Ya dentro del contenedor:

```bash
antlr -Dlanguage=Python3 MiniLang.g4
```

Esto genera `MiniLangLexer.py`, `MiniLangParser.py`, `MiniLangListener.py` y las tablas internas `MiniLang.tokens` / `.interp`.

### 3. Correr el analizador

```bash
python3 Driver.py program_test_correct.txt
python3 Driver.py program_test_error.txt
```

- Si el archivo es sintácticamente correcto, no se imprime nada — el silencio significa éxito.
- Si hay errores, ANTLR los reporta indicando línea, posición y token problemático.

## Análisis de la gramática (`MiniLang.g4`)

Un archivo `.g4` se divide en dos tipos de reglas: las **sintácticas** (minúsculas: `prog`, `stat`, `expr`), que definen la estructura del lenguaje, y las **léxicas** (MAYÚSCULAS: `ID`, `INT`, `NEWLINE`, `MUL`, `DIV`, `ADD`, `SUB`, `WS`), que definen cómo se reconocen los tokens.

```antlr
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
```

- `prog` es la regla raíz: un programa es una o más instrucciones.
- `stat` puede ser una expresión que se imprime, una asignación de variable, o una línea vacía.
- `expr` define expresiones aritméticas con paréntesis, operadores y recursividad.

**Uso de `#`:** cada alternativa dentro de una regla puede etiquetarse con `#nombre`. Esto le indica a ANTLR que genere un método distinto para cada caso al implementar un Visitor o Listener, en vez de agrupar todo en un solo método genérico.

**Precedencia de operadores:** se controla por el orden de las alternativas. Como `*` y `/` están antes que `+` y `-`, tienen mayor precedencia, igual que en matemáticas.

Reglas léxicas:

```antlr
MUL : '*' ;
DIV : '/' ;
ADD : '+' ;
SUB : '-' ;
ID  : [a-zA-Z]+ ;
INT : [0-9]+ ;
NEWLINE : '\r'? '\n' ;
WS  : [ \t]+ -> skip ;
```

`WS -> skip` descarta espacios y tabulaciones automáticamente, para que el parser nunca tenga que lidiar con ellos.

## Análisis de `Driver.py`

```python
input_stream = FileStream(argv[1])   # Lee el archivo de entrada
lexer = MiniLangLexer(input_stream)  # Convierte el texto en tokens
stream = CommonTokenStream(lexer)    # Almacena el flujo de tokens
parser = MiniLangParser(stream)      # Crea el parser
tree = parser.prog()                 # Inicia el análisis desde la regla raíz
```

El `Driver.py` conecta todas las piezas: abre el archivo, lo pasa por el lexer para tokenizarlo, agrupa los tokens en un `CommonTokenStream`, y arranca el análisis sintáctico desde la regla `prog`, generando el árbol de sintaxis abstracta.

## Casos de prueba

| Archivo | Resultado | Explicación |
|---|---|---|
| `program_test_correct.txt` | Sin salida (éxito) | El programa respeta la gramática definida |
| `program_test_error.txt` | `line 5:2 mismatched input '=' expecting NEWLINE` | Contiene `4 = c`, una asignación inválida: el lado izquierdo del `=` debe ser un `ID`, no un `INT` |

## Video de demostración

https://youtu.be/rA1eLOVz700


En el video se muestra la estructura del proyecto, la construcción del contenedor Docker, la generación del lexer/parser con ANTLR, y una prueba exitosa junto con una prueba con error.
