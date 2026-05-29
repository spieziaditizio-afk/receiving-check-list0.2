# CLAUDE.md

> Guía para Claude Code sobre este proyecto. El **código y la UI van en inglés**; este documento está en español porque es el idioma de trabajo con el desarrollador.

## Qué es

Web app que ayuda a **conciliar el packing list (en papel) contra los pallets y cajas físicas** que llegan al almacén, y a generar las etiquetas de identificación de cada pallet. Reemplaza el proceso actual, que se hace **visualmente caja por caja y con calculadora**.

App de estación: corre en el navegador de los PCs de la zona *receiving*. Pensada para uso intensivo con scanner; la velocidad y el flujo sin fricción son el objetivo principal.

## Contexto del negocio

- **Dónde:** almacén de **Arrow Electronics**, departamento de **Microsoft**, ubicado en **Sevenum (Países Bajos)**.
- **Proceso:** los envíos llegan en pallets con un packing list en papel. Se verifica **PO, Delivery Number, Part Number (PN), COO (Country of Origin)** y **cantidades de piezas**.
- **Reorganización:** al llegar, las cajas se ordenan en pallets de modo que **cada pallet contenga cajas de un solo PO + un solo Delivery + un solo PN + un solo COO**. Luego se registran los pallets en el **WMS** y se imprimen las **etiquetas adhesivas** de identificación para el storage.

## Glosario de dominio

| Término | Significado |
|---|---|
| **PO** | Purchase Order. Un proveedor puede enviar un PO con 1 o más deliveries. |
| **Delivery (Number)** | Entrega dentro de un PO. Puede tener 1 o más PN y 1 o más COO. |
| **PN** | Part Number. |
| **COO** | Country of Origin. |
| **Packing List** | Documento en papel que acompaña al envío. **Caso normal: 1 PO, 1 Delivery, 1 PN, 1 COO.** Pero hay que soportar los casos compuestos (varios deliveries / PN / COO). |
| **Pallet** | Tarima física. Regla: 1 PO + 1 Delivery + 1 PN + 1 COO por pallet. |
| **Caja** | Caja cerrada con etiqueta de barcode escaneable y su cantidad de piezas. |
| **WMS** | Sistema de gestión de almacén de Arrow (externo a esta app). |

## Reglas de negocio (invariantes)

1. **Un pallet = 1 PO + 1 Delivery + 1 PN + 1 COO.** La app no debe permitir mezclar.
2. **Cajas cerradas:** no se abren ni se recuentan piezas a mano. Se despacha con la cantidad que indica la **etiqueta de cada caja**.
3. La conciliación es: **suma de piezas escaneadas (por caja) vs. total declarado en el packing list**, por cada combinación PO/Delivery/PN/COO.
4. Un packing list puede repartirse en **varios pallets**; la app agrupa y segrega por pallet.

## Modelo de datos y conciliación

### Packing list (ingreso manual, jerárquico)
El packing list se registra una vez respetando la jerarquía real:

```
PO
└─ Delivery (1+ por PO)
   └─ Línea: PN + COO + cantidad declarada (1+ por delivery)
```

Esto es lo que se concilia contra lo escaneado.

### Dos niveles de "dato master"
Los proveedores varían en qué datos imprimen en la etiqueta de caja. Para no preguntar campo por campo, se usan dos masters:

1. **Scan profile (por proveedor)** — define *qué campos trae la etiqueta de caja* de ese proveedor. Casos típicos:
   - **Completo:** PO, Delivery, PN, COO, qty (se escanea todo).
   - **Habitual (mayoría):** PN + COO + qty (PO y Delivery se heredan del master).
   - **Mínimo:** solo qty (el resto se hereda).

   Se configura una vez por proveedor y queda como dato persistente.

2. **Contexto del pallet** — como la regla es **1 PO + 1 Delivery + 1 PN + 1 COO por pallet**, esos 4 valores son constantes para todas las cajas del pallet. El trabajador los fija una vez (eligiéndolos del packing list ya cargado) y quedan fijos para ese pallet.

### Lógica de escaneo por caja
- **Solo se escanea lo que la etiqueta realmente trae.** Lo que no trae se **hereda del contexto del pallet**.
- **La cantidad de piezas se escanea SIEMPRE** (es el núcleo de la conciliación).
- **Validación:** si un campo *sí* viene en el barcode (p. ej. PN), se compara contra el master del pallet. Mismatch → alerta de **"caja en pallet equivocado"**. Si no viene, se rellena del master sin fricción.
- **Principio:** escanear siempre que la etiqueta lo tenga; inferir del master solo lo que físicamente no está impreso. Cada campo heredado deja de verificarse por caja (se confía en el ordenado físico + la conciliación de piezas atrapa errores de cantidad).

## Stack y arquitectura

- **React + Vite + TypeScript** (SPA). Sin backend por ahora — **app autónoma, ingreso manual de datos** (el packing list de papel se teclea/escanea). No hay integración con el WMS en esta fase.
- Estado en memoria/local; persistencia local (p. ej. `localStorage`/IndexedDB) si se necesita conservar una sesión de verificación. Confirmar antes de añadir dependencias pesadas.

## Requisitos funcionales clave

### Flujo de escaneo (lo más importante)
- El scanner **ZEBRA DS3678** funciona como **keyboard wedge (HID)**: "teclea" el contenido del barcode y emite Enter/Tab al final.
- **Auto-advance del cursor:** al completar el escaneo de una caja, el foco debe pasar **automáticamente al siguiente campo/fila** sin que el trabajador toque el mouse ni el teclado. Cero clics manuales durante el escaneo en serie.
- **Limpieza de cantidades escaneadas:** las etiquetas de caja pueden traer la cantidad con **ceros a la izquierda y/o letras** (p. ej. `0000150`, `150PCS`, `PCS150`). La app debe **extraer el entero limpio** (→ `150`) antes de sumar.

### Conciliación
- Comparar piezas escaneadas vs. packing list por PO/Delivery/PN/COO (ver **Modelo de datos y conciliación**). Mostrar claramente **OK / faltante / sobrante**.

### Salidas imprimibles (etiquetas ZPL)
Dos tipos de "label ticket", impresos en la misma etiqueta física:

1. **Resumen del packing list, segregado por pallet:** por cada pallet → PO, Delivery, PN, COO, **cantidad de cajas** y **cantidad total de piezas**.
2. **Ticket por pallet con desglose por caja:** lista las cajas del pallet y las piezas de cada una. Objetivo: que en el **picking** futuro se sepan las cajas y sus cantidades **sin bajar el pallet al suelo**.

## Hardware y entorno (restricciones)

- **Estaciones móviles** en zona *receiving*: PC + monitor + scanner.
- **Scanner:** ZEBRA barcode **DS3678** (modo HID/keyboard wedge).
- **Impresora:** ZEBRA serie **ZT** (industrial). Imprimir con **ZPL directo** (no print del navegador) para control exacto.
- **Etiqueta:** **12,8 cm × 6,3 cm**. Verificar la **densidad de la impresora (203 vs 300 dpi)** al definir las plantillas ZPL — a 203 dpi (8 dots/mm) son ~1024 × 504 dots.

## Convenciones

- **UI y código en inglés** (variables, comentarios, textos de pantalla).
- Priorizar **velocidad de flujo y robustez del escaneo** sobre features extra.
- Mantener plantillas ZPL separadas y parametrizadas (no hardcodear datos).

## Decisiones pendientes

- Cómo enviar el ZPL a la Zebra desde el navegador (driver de Windows + raw print, Zebra Browser Print, o endpoint local). **Confirmar antes de implementar la impresión.**
- DPI exacto de las ZT en uso (203 vs 300) para dimensionar las plantillas.
- Si se necesita persistir/historizar sesiones de verificación o basta con conciliación en vivo.
- Si se precarga una **lista fija de proveedores con su scan profile ya conocido**, o el perfil se define la primera vez que aparece cada proveedor.
