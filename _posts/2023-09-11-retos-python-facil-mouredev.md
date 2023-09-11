---
layout: single
title: Retos Semanales Faciles - Mouredev
excerpt: "Resolución de todos los retos semanales de 2023 de programacion en Python de Mouredev de nivel fácil."
date: 2023-09-11
classes: wide
header:
  teaser: /assets/images/retos-mouredev/retos-python_easy.png
  teaser_home_page: true
  icon: /assets/images/python.png
categories:
  - Python
tags:
  - Retos
  - Python
---

![](/assets/images/retos-mouredev/retos-python_easy.png)

# Índice

1. [EL FAMOSO "FIZZ BUZZ"](#el-famoso-fizz-buzz)
2. [EL "LENGUAJE HACKER"](#el-lenguaje-hacker)
3. [HETEROGRAMA, ISOGRAMA Y PANGRAMA](#heterograma-isograma-y-pangrama)
4. [URL PARAMS](#url-params)
5. [VIERNES 13](#viernes-13)
6. [OCTAL Y HEXADECIMAL](#octal-y-hexadecimal)
7. [AUREBESH](#aurebesh)
8. [CIFRADO CÉSAR](#cifrado-cesar)
9. [EL CARÁCTER INFILTRADO](#el-caracter-infiltrado)
10. [EL ÁBACO](#el-abaco)

## EL FAMOSO "FIZZ BUZZ"

```python
#!/usr/bin/python3

"""
  Escribe un programa que muestre por consola (con un print) los
  números de 1 a 100 (ambos incluidos y con un salto de línea entre
  cada impresión), sustituyendo los siguientes:
    - Múltiplos de 3 por la palabra "fizz".
    - Múltiplos de 5 por la palabra "buzz".
    - Múltiplos de 3 y de 5 a la vez por la palabra "fizzbuzz".
 """

def fizzbuzz():
    for num in range(1,101):
        if num % 3 == 0 and num % 5 == 0:
            print('fizzbuzz')
        elif num % 3 == 0:
            print('fizz')
        elif num % 5 == 0:
            print('buzz')
        else:
            print(num)

fizzbuzz()
```
<span style="float: right;"><a href="#Indice">Volver al índice</a></span>

## EL "LENGUAJE HACKER"

```python
#!/usr/bin/python3

"""
    Escribe un programa que reciba un texto y transforme lenguaje natural a
    "lenguaje hacker" (conocido realmente como "leet" o "1337"). Este lenguaje
    se caracteriza por sustituir caracteres alfanuméricos.
        - Utiliza esta tabla (https://www.gamehouse.com/blog/leet-speak-cheat-sheet/) 
        con el alfabeto y los números en "leet".
        (Usa la primera opción de cada transformación. Por ejemplo "4" para la "a")
"""

def leet_translator(text):
    
    leet_dict = {
    'A': '4','B': 'I3','C': '[','D': ')','E': '3','F': '|=','G': '&','H': '#','I': '1','J': ',_|',
    'K': '>|','L': '1','M': '|\\/|','N': '^/','O': '0','P': '|*','Q': '(_,)','R': 'I2','S': '5',
    'T': '7','U': '(_)','V': '\\/','W': '\\/\\/','X': '><','Y': 'j','Z': '2',
    '0': 'o','1': 'L','2': 'R','3': 'E','4': 'A','5': 'S','6': 'b','7': 'T','8': 'B','9': 'g'
    }
    text_leet = ""
    
    for character in text.upper():
        if character in leet_dict:
            text_leet += leet_dict.get(character) # leet_dict[character]
        else:
            text_leet += character
    return text_leet

print(leet_translator("Leet"))
print(leet_translator("Aquí está un texto de prueba para ver si funciona el reto!"))
print(leet_translator(input("Texto a traducir: ")))
```
<span style="float: right;"><a href="#Indice">Volver al índice</a></span>

## HETEROGRAMA, ISOGRAMA Y PANGRAMA

```python
"""
/*
 * Crea 3 funciones, cada una encargada de detectar si una cadena de
 * texto es un heterograma, un isograma o un pangrama.
 * - Debes buscar la definición de cada uno de estos términos.
 */

 Problemas con mi primera solución:
 Heterograma: No funciona bien con espacios o caracteres especiales
 Isograma: ok
 Pangrama: No funciona correctamente con caracteres especiales
 Otros: Hay mucho código repetido que se puede simplicar (creando una funcion que cuente las letras)
 """

import re

from unidecode import unidecode # Con esto pasaremos todo a unidecode

def __char_count(text: str) -> dict[str, int]:
    text_no_numbers = re.sub(r'[0-9]', '', text.lower().replace(' ','')) # r"\d" otra forma para eliminar dígitos
    text_no_punt = re.sub(r"[^\w\s]", '', text_no_numbers)
  
    # Obtenemos el unicode pero preservando la ñ
    text_unicode = unidecode(text_no_punt.replace("ñ", ".")).replace(".", "ñ")
    
    char_dict = dict()
       
    for char in text_unicode:
        if char in char_dict.keys():
            char_dict[char] += 1
        else:
            char_dict[char] = 1
    return char_dict

def is_heterogram(text: str) -> bool:
    for counter in __char_count(text).values():
        if counter > 1:
            return False
    return True

def is_isogram(text: str) -> bool:
    principal_counter = 0
    for counter in __char_count(text).values():
        if principal_counter == 0:
            principal_counter = counter
        if principal_counter is not counter:
            return False
    return True

def is_pangram(text: str) -> bool:
    return len(__char_count(text).keys()) == 27


print(is_heterogram("hiperblanduzcos"))
print(is_heterogram("hiperblanduzcós    !!w"))
print(is_isogram("anna"))
print(is_pangram("Benjamín pidió una bebida de kiwi y fresa. Noé, sin vergüenza, la más exquisita champaña del menú"))
```
<span style="float: right;"><a href="#Indice">Volver al índice</a></span>

## URL PARAMS

```python
"""
/*
 * Dada una URL con parámetros, crea una función que obtenga sus valores.
 * No se pueden usar operaciones del lenguaje que realicen esta tarea directamente.
 *
 * Ejemplo: En la url https://retosdeprogramacion.com?year=2023&challenge=0&test=1
 * los parámetros serían ["2023", "0"]
 */
"""

import re

def get_url_params(url: str) -> list[str]:
    url_list = list()
    url_components = url.split("&")
    for component in url_components:
        if "=" in component:
            param = component.split("=")
            if len(param) == 2 and param[1] != "":
                url_list.append(param[1])
    return url_list

def get_url_params_v2(url: str) -> list[str]:
    new_url = re.split(r'[=&]', url)
    url_list = list()
    for i in range(1,len(new_url), 2):
        if new_url[i] != "":
            url_list.append(new_url[i])
    return url_list

def get_url_params_regular_exp(url: str) -> list[str]:
    regex = r'=([a-zA-Z0-9._%-]+)'
    params = re.findall(regex, url)
    return params

url = "https://retosdeprogramacion.com?year=2023&challenge=&languaje="

url_params = get_url_params(url)
print(url_params)
url_params = get_url_params_v2(url)
print(url_params)
url_params = get_url_params_regular_exp(url)
print(url_params)
```
<span style="float: right;"><a href="#Indice">Volver al índice</a></span>

## VIERNES 13

```python
"""
/*
 * Crea una función que sea capaz de detectar si existe un viernes 13 en el mes y el año indicados.
 * - La función recibirá el mes y el año y retornará verdadero o falso.
 */
"""

import datetime

MONTH_DICT = {
    'Enero': 1, 'Febrero': 2, 'Marzo': 3, 'Abril': 4, 'Mayo': 5, 'Junio': 6, 'Julio': 7,
    'Agosto': 8, 'Septiembre': 9, 'Octubre': 10, 'Noviembre': 11, 'Diciembre': 12
    }

def is_friday_13(month: str,year: int) -> bool:
    day_13 = 13
    number_month = MONTH_DICT.get(month)
    try:
        return datetime.date(year,number_month,day_13).weekday() == 4
    except:
        return False      
        
year = int(input("Dime el año: "))
month = input("Dime el mes: ")

if month not in MONTH_DICT.keys():
    print("Error. Mes incorrecto")
else:
    if is_friday_13(month, year):
        print("El mes %s del año %d tiene un Viernes 13." % (month,year))
    else:
        print("El mes %s del año %d no tiene un Viernes 13."% (month, year))
```
<span style="float: right;"><a href="#Indice">Volver al índice</a></span>

## OCTAL Y HEXADECIMAL

```python
"""
/*
 * Crea una función que reciba un número decimal y lo trasforme a Octal
 * y Hexadecimal.
 * - No está permitido usar funciones propias del lenguaje de programación que
 * realicen esas operaciones directamente.
 */
"""

def convert_octal(number: int) -> str:
    octal_number = ""
    while number != 0:
        cociente = int(number / 8)
        residuo = number % 8
        octal_number += str(residuo)
        number = cociente
    return octal_number[::-1]
        
def convert_hexadecimal(number: int) -> str:
    tuple_hexadecimal = (0,1,2,3,4,5,6,7,8,9,"A","B","C","D","E","F")
    hexadecimal_number = ""
    while number != 0:
        cociente = int(number / 16)
        residuo = number % 16
        hexadecimal_number += str(tuple_hexadecimal[residuo])
        number = cociente
    return hexadecimal_number[::-1]

print(convert_hexadecimal(7000))
```
<span style="float: right;"><a href="#Indice">Volver al índice</a></span>

## AUREBESH

```python
"""
/*
 * Crea una función que sea capaz de transformar Español al lenguaje básico del universo
 * Star Wars: el "Aurebesh".
 * - Puedes dejar sin transformar los caracteres que no existan en "Aurebesh".
 * - También tiene que ser capaz de traducir en sentido contrario.
 *  
 * ¿Lo has conseguido? Nómbrame en twitter.com/mouredev y escríbeme algo en Aurebesh.
 *
 * ¡Que la fuerza os acompañe!
 */

 # Mi solución no contemplaba caracteres dobles por lo que lo he tenido que cambiar
 ayudandome del ejemplo de Mouredev
"""
from unidecode import unidecode

def convert_aurebesh(text: str, aurebesh: bool) -> str:
    basic_dict = {
        "a": "aurek", "b": "besh", "c": "cresh", "d": "dorn", "e": "esk", "f": "forn", "g": "grek", "h": "herf",
        "i": "isk", "j": "jenth", "k": "krill", "l": "leth", "m": "merm", "n": "nern", "o": "osk", "p": "peth", "q": "qek",
        "r": "resh", "s": "senth", "t": "trill", "u": "usk", "v": "vev", "w": "wesk", "x": "xesh", "y": "yirt", "z": "zerek",
        "ae": "enth", "eo": "onith", "kh": "krenth", "ng": "nen", "oo": "orenth", "sh": "sen", "th": "thesh"}
    
    aurebesh_dict = dict()
    # Invertir el dicciconario
    for key,value in basic_dict.items():
        aurebesh_dict[value] = key
    
    # Añadir la ñ al diccionario
    unicode_text = unidecode(text.lower().replace("ñ","{?}")).replace("{?}","ñ")
    translated_text = ""

    # Condicionales para gestionar si esta en aurebesh o no
    if aurebesh:
        translated_text = text
        for key, value in aurebesh_dict.items():
            translated_text = translated_text.replace(key,value)
    else:
        text_len = len(text)
        i = 0
        # Cambiado del for para evitar repetidas con los caracteres dobles
        while i < text_len:
            one_character = unicode_text[i]
            double_character = ""
            # Sacar la primera combinacion de 2 caracteres
            if i < text_len -1:
                double_character = one_character + unicode_text[i +1]
            # Comprobar si esa combinacion esta en el diccionario. Si está se guarda en el texto
            if double_character in basic_dict:
                translated_text += basic_dict[double_character]
                i += 2
            # Guardar el caracter en el texto si está en el diccionario
            else:
                translated_text += basic_dict[one_character] if one_character in basic_dict else one_character
                i += 1
    return translated_text

# Comprobaciones
aurebesh = convert_aurebesh("The MoureDev", False)
print(aurebesh)
basic = convert_aurebesh(aurebesh, True)
print(basic)

aurebesh = convert_aurebesh("Qué te ha parecido el reto? A mí me ha gustado mucho! Mañana sigue practicando.", False)
print(aurebesh)
basic = convert_aurebesh(aurebesh, True)
print(basic)
```
<span style="float: right;"><a href="#Indice">Volver al índice</a></span>

## CIFRADO CESAR

```python
"""
/*
 * Crea un programa que realize el cifrado César de un texto y lo imprima.
 * También debe ser capaz de descifrarlo cuando así se lo indiquemos.
 *
 * Te recomiendo que busques información para conocer en profundidad cómo
 * realizar el cifrado. Esto también forma parte del reto.
 */
"""

import string

def cesar_cipher(text: str, decrypt = False, jump = 3) -> str:
    alphabet = list(string.ascii_lowercase)
    alphabet.insert(alphabet.index("n")+1,"ñ")
    cesar_text = ""
    for char in text.lower():
        if char in alphabet:
            index = (alphabet.index(char) + (-jump if decrypt else jump)) % len(alphabet)
            cesar_text += alphabet[index]
        else:
            cesar_text += char
    return cesar_text

print(cesar_cipher("Mi nombre es MoureDev."))
print(cesar_cipher("ol proeuh hv orxuhghy.", True))
print(cesar_cipher("Mi nombre es MoureDev.", jump=5))
print(cesar_cipher("qn rtqgwj jx qtzwjija.", True, 5))
```
<span style="float: right;"><a href="#Indice">Volver al índice</a></span>

## EL CARACTER INFILTRADO

```python
"""
/*
 * Crea una función que reciba dos cadenas de texto casi iguales,
 * a excepción de uno o varios caracteres. 
 * La función debe encontrarlos y retornarlos en formato lista/array.
 * - Ambas cadenas de texto deben ser iguales en longitud.
 * - Las cadenas de texto son iguales elemento a elemento.
 * - No se pueden utilizar operaciones propias del lenguaje
 *   que lo resuelvan directamente.
 * 
 * Ejemplos:
 * - Me llamo mouredev / Me llemo mouredov -> ["e", "o"]
 * - Me llamo.Brais Moure / Me llamo brais moure -> [" ", "b", "m"]
 */
"""

def caracter_infiltrado(text_a: str, text_b: str) -> list():
    list_desigual = list()
    if len(text_a) == len(text_b):
        for char_1, char_2 in zip(text_a, text_b):
            if char_1 != char_2:
                list_desigual.append(char_2)       
    else:
        return list_desigual
    return list_desigual

print(caracter_infiltrado("Me llamo mouredev", "Me llemo mouredov"))
print(caracter_infiltrado("Me llamo.Brais Moure", "Me llamo brais moure"))
```
<span style="float: right;"><a href="#Indice">Volver al índice</a></span>

## EL ABACO 

```python
"""
/*
 * Crea una función que sea capaz de leer el número representado por el ábaco.
 * - El ábaco se representa por un array con 7 elementos.
 * - Cada elemento tendrá 9 "O" (aunque habitualmente tiene 10 para realizar operaciones)
 *   para las cuentas y una secuencia de "---" para el alambre.
 * - El primer elemento del array representa los millones, y el último las unidades.
 * - El número en cada elemento se representa por las cuentas que están a la izquierda del alambre.
 *
 * Ejemplo de array y resultado:
 * ["O---OOOOOOOO",
 *  "OOO---OOOOOO",
 *  "---OOOOOOOOO",
 *  "OO---OOOOOOO",
 *  "OOOOOOO---OO",
 *  "OOOOOOOOO---",
 *  "---OOOOOOOOO"]
 *  
 *  Resultado: 1.302.790
 */
"""

def read_abaco():
    abaco_list = [
        "O---OOOOOOOO",
        "OOO---OOOOOO",
        "---OOOOOOOOO",
        "OO---OOOOOOO",
        "OOOOOOO---OO",
        "OOOOOOOOO---",
        "---OOOOOOOOO"]
    
    count_ceros = 0
    count_ceros_dict = list()

    # Variable para pasar la lista a entero
    real_number = 0
    # Contamos las O de la izquierda
    for filas in abaco_list:
        for values in filas:
            if values == "O":
                count_ceros += 1
            if values == "-":
                break
        count_ceros_dict.append(count_ceros)
        count_ceros = 0
    # Transformamos el array en un int:
    for current_digit in count_ceros_dict:
        real_number = real_number*10 + current_digit
    # Formateamos numero para poner ,
    real_number = f'{real_number:,}'

    print(real_number.replace(",","."))
    
read_abaco()
```
<span style="float: right;"><a href="#Indice">Volver al índice</a></span>
