# **Construcción de Compiladores - Laboratorio 2**

[**Derek Arreaga**](https://github.com/FabianKel)
- **Repositorio**: https://github.com/FabianKel/COMPIS-LAB2
> [Video de explicación]('https://youtu.be/CV4txFR7kLg')
---
## 1. ¿Por qué el archivo *program\_test\_pass.txt* sí pasa?

### Contenido del archivo:

```txt
5 + 3.0
10 * 2.5
4.5 + 2
```

### Análisis:

* Todos los *tipos* involucrados son `int` y `float`.
* Operaciones `+` y `*` están definidas para esos tipos, y las combinaciones de tipos que se aceptan son:

  * `int + float` → `float`
  * `int * float` → `float`
  * `float + int` → `float`

---

## 2. ¿Por qué el archivo *program\_test\_fail.txt* no pasa?

### Contenido del archivo:

```txt
8 / "2.0"
(3 + 2) * "7"
9.0 - true
"hello" + 3
```

Al ejecutarlo, se obtienen estos errores:

> Type checking error: Unsupported operand types for \* or /: int and string
> Type checking error: Unsupported operand types for \* or /: int and string
> Type checking error: Unsupported operand types for + or -: float and bool
> Type checking error: Unsupported operand types for + or -: string and int

<center>

| Línea           | Error detectado |
| --------------- | --------------- |
| `8 / "2.0"`     | `int / string`  |
| `(3 + 2) * "7"` | `int * string`  |
| `9.0 - true`    | `float - bool`  |
| `"hello" + 3`   | `string + int`  |

</center>

Estas combinaciones de tipos y operaciones no están permitidas en la lógica de validación de tipos.

---

## 3. Extender la gramática con 2 operaciones nuevas

Las operaciones que decidí agregar son:

* **Módulo: `%`**
* **Potencia: `^` o `**`**

Para ello, se actualizó la regla `expr` en `SimpleLang.g4`:

```antlr
expr: expr op=('^'|'**')  expr     # Pow
    | expr op='%'         expr     # Mod
    | expr op=('*'|'/')   expr     # MulDiv
    | expr op=('+'|'-')   expr     # AddSub
    | INT                          # Int
    | FLOAT                        # Float
    | STRING                       # String
    | BOOL                         # Bool
    | '(' expr ')'                 # Parens
    ;
```

Y además se declararon los tokens en la sección de abajo así:

```antlr
MOD: '%' ;
POW: '^'|'**' ;
```

Después de modificar la gramática, se **regeneraron el parser y lexer** dentro del contenedor con:

```bash
antlr -Dlanguage=Python3 -visitor SimpleLang.g4
antlr -Dlanguage=Python3 -listener SimpleLang.g4
```

Si se modifica el `program_test_pass.txt` así:

```text
5 ** 2
5 mod 3
4 % 2
5^2

```

Al ejecutar los Drivers, se muestra que las cadenas son aceptadas. Aún así, todavía nos queda agregar las reglas de tipos para estas operaciones. Para ello, se modificaron los archivos `type_check_listener.py` y `type_check_visitor.py` respectivamente con:

```python
  def exitPow(self, ctx: SimpleLangParser.PowContext):
    left_type = self.types[ctx.expr(0)]
    right_type = self.types[ctx.expr(1)]

    if isinstance(left_type, (IntType, FloatType)) and isinstance(right_type, (IntType, FloatType)):
      self.types[ctx] = FloatType() if isinstance(left_type, FloatType) or isinstance(right_type, FloatType) else IntType()
    else:
      self.errors.append(f"Unsupported operand types for ^: {left_type} and {right_type}")

  def exitMod(self, ctx: SimpleLangParser.ModContext):
    left_type = self.types[ctx.expr(0)]
    right_type = self.types[ctx.expr(1)]

    if isinstance(left_type, IntType) and isinstance(right_type, IntType):
      self.types[ctx] = IntType()
    else:
      self.errors.append(f"Unsupported operand types for %: {left_type} and {right_type}")

```

```python
  def visitPow(self, ctx: SimpleLangParser.PowContext):
    left_type = self.visit(ctx.expr(0))
    right_type = self.visit(ctx.expr(1))

    if isinstance(left_type, (IntType, FloatType)) and isinstance(right_type, (IntType, FloatType)):
      return FloatType() if isinstance(left_type, FloatType) or isinstance(right_type, FloatType) else IntType()
    else:
      raise TypeError(f"Unsupported operand types for ^: {left_type} and {right_type}")

  def visitMod(self, ctx: SimpleLangParser.ModContext):
    left_type = self.visit(ctx.expr(0))
    right_type = self.visit(ctx.expr(1))

    if isinstance(left_type, IntType) and isinstance(right_type, IntType):
      return IntType()
    else:
      raise TypeError(f"Unsupported operand types for %: {left_type} and {right_type}")

```

Las reglas definidas, representan lo siguiente:
- **``POW``**: solo se permite entre **``INT``** o **``FLOAT``**

- **``MOD``**: solo se permite entre **``INT``**
---

### Explicación del uso de Visitor vs Listener:

* **Listener**: ANTLR lo llama automáticamente por cada nodo, es más pasivo e ideal para recolectar información.
* **Visitor**: Se controla el recorrido, siendo más flexible eideal para evaluación y chequeo semántico complejo.
