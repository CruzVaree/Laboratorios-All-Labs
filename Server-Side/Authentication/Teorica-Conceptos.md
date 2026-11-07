# 🔐 Vulnerabilidades de Autenticación (Authentication)

## 📌 00 - Fundamentos

### Autenticación vs. Autorización
- **Autenticación**: responde "¿quién eres?". Es el proceso de verificar la identidad de un usuario, normalmente con usuario + contraseña.
- **Autorización**: responde "¿qué puedes hacer?". Una vez que el sistema sabe quién eres, decide qué recursos o acciones te están permitidos.
- Son conceptos **independientes**. Muchas vulnerabilidades nacen de confundirlos o de implementarlos mal por separado (ej. un usuario autenticado accede a datos que no debería ver = fallo de autorización, no de autenticación).

### Factores de autenticación
| Factor | Descripción | Ejemplo |
|---|---|---|
| Algo que **sabes** | Conocimiento secreto | Contraseña, PIN |
| Algo que **tienes** | Posesión física/digital | Token, app generadora de códigos, SMS |
| Algo que **eres** | Biometría | Huella, reconocimiento facial |

- **MFA (Multi-Factor Authentication)**: combina 2 o más factores. La seguridad depende de *cómo* se combinan, no solo de que existan. Ahí es donde suelen aparecer los fallos.

### Flujo típico de inicio de sesión
1. El usuario envía `usuario` + `contraseña`.
2. El servidor compara la contraseña contra el **hash** almacenado (nunca se guarda en texto plano).
3. Si coincide, el servidor crea una **sesión**.
4. Esa sesión se mantiene con una **cookie** o **token**, que el navegador envía automáticamente en cada petición posterior.
5. El servidor valida esa cookie/token en cada request para saber "sigues siendo tú" sin pedir la contraseña de nuevo.

### Dónde vive el estado de sesión
- **Cookies de sesión** (`Set-Cookie` en la respuesta HTTP).
- **Tokens JWT** (JSON Web Tokens).
- **Almacenamiento de sesión del lado del servidor** (server-side session storage).

### Por qué fallan estos sistemas (causa raíz)
> La gran mayoría de los fallos de autenticación **no son "magia de hacker"**: son errores de lógica de negocio.

Causas típicas:
- El desarrollador no contempló un **caso límite**.
- Se confió en información que viene **del lado del cliente** (manipulable por el atacante).
- No se **limitó** cuántas veces se puede intentar algo (sin rate limiting).

**Mentalidad clave para explotar estos fallos:** preguntarte constantemente *"¿qué caso no contempló quien programó esto?"*.

---

## 🛠️ 01 - Técnicas

### 1. Fuerza bruta de credenciales (Brute-force)
**Qué es:** probar múltiples combinaciones de contraseña contra un usuario conocido, o probar listas completas de usuario+contraseña filtradas (**credential stuffing**).

**Qué observar en un laboratorio:**
- ¿Existe bloqueo de cuenta tras X intentos fallidos?
- ¿Ese bloqueo se puede **evadir**?
  - Rotando el header `X-Forwarded-For`
  - Cambiando el `User-Agent`
  - Alternando entre varios usuarios para no disparar el límite en ninguno en particular (ataque "ancho" en vez de "profundo")

---

### 2. Enumeración de usuarios (Username Enumeration)
**Qué es:** la aplicación revela, sin querer, si un nombre de usuario existe en el sistema.

**Formas en que se filtra esta información:**
- **Mensajes de error distintos**: "usuario no existe" vs. "contraseña incorrecta".
- **Código de respuesta HTTP diferente** entre un usuario válido e inválido.
- **Timing attack**: aunque el mensaje sea idéntico, el servidor tarda más en responder cuando el usuario SÍ existe (porque llega hasta el paso de verificar el hash de la contraseña), y responde más rápido cuando el usuario NO existe.

> El timing attack es la variante más elegante porque no depende de ningún mensaje visible.

---

### 3. Lógica de inicio de sesión mal implementada (Broken Login Logic)
Ejemplos típicos en laboratorios:
- Función de **"recordarme"** que no vuelve a pedir la contraseña cuando debería.
- Proceso de login **multi-paso** (contraseña → 2FA) donde se puede **saltar el segundo paso** navegando directamente a la URL de "sesión iniciada".
- Formulario de **cambio de contraseña** que no verifica cuál es la contraseña actual antes de permitir el cambio.

---

### 4. Fallos en la autenticación multifactor (MFA/2FA)
- El código de verificación (4-6 dígitos) **no tiene límite de intentos** → vulnerable a fuerza bruta, igual que una contraseña débil.
- El servidor **confía en un parámetro del cliente** (ej. una cookie o campo oculto) para decidir si el segundo factor ya se completó → ese parámetro se puede modificar manualmente.

---

### 5. Tokens de sesión débiles
- Tokens **predecibles**: secuenciales, o derivados directamente de un timestamp.
- **JWT mal validado**:
  - El servidor acepta tokens firmados con algoritmo `"none"`.
  - No se verifica correctamente la firma del token.
  - Esto permite **falsificar tokens completos**.

---

### 6. Restablecimiento de contraseña mal implementado (Password Reset)
- Token de reseteo **adivinable** (patrón predecible).
- Token que **no expira nunca**.
- Token que **sigue siendo válido** incluso después de haberse usado una vez (no se invalida tras el uso).

---

## 🧰 Herramientas para practicar

| Herramienta | Uso |
|---|---|
| **Burp Suite - Intruder** | Automatizar fuerza bruta (contraseñas, tokens, usuarios) |
| **Burp Suite - Repeater** | Manipular peticiones manualmente, probar lógica paso a paso, saltar pasos del flujo |

---

## ✅ Checklist mental antes de atacar un login

- [ ] ¿Hay diferencia en mensajes de error entre usuario válido/inválido?
- [ ] ¿Hay diferencia de tiempos de respuesta?
- [ ] ¿Existe límite de intentos? ¿Se puede evadir?
- [ ] ¿El flujo tiene varios pasos? ¿Se puede saltar alguno?
- [ ] ¿El 2FA tiene límite de intentos?
- [ ] ¿El token de sesión/reset es predecible?
- [ ] ¿Se puede manipular un parámetro del cliente para alterar el resultado?
