# Usuarios, grupos, roles y privilegios

Plantilla de Zumbaruta en Odoo 19, diseñada con el **principio de mínimo privilegio**: cada persona solo puede hacer lo que necesita su puesto.

> Aquí solo aparece el correo de acceso de cada usuario. Las contraseñas no están en el repositorio.

## Cómo se organiza la seguridad en Odoo 19

| Nivel | Qué decide | Dónde se ve | Ejemplo |
|---|---|---|---|
| **Rol** | El tipo de usuario | Ficha del usuario → Roles | Usuario, Administrador |
| **Privilegio** | La opción de cada módulo en la ficha del usuario | Ajustes → Usuarios y compañías → Privilegios | Ventas, Contabilidad |
| **Grupo** | El conjunto de permisos | Ajustes → Usuarios y compañías → Grupos | Usuario: solo mostrar documentos propios |
| **Derecho de acceso** | Qué se puede hacer con un modelo: L/E/C/B | Grupo → Permisos de acceso | `sale.order`: L E C |
| **Regla de registro** | Sobre qué registros se aplica | Grupo → Reglas de registro | Solo sus clientes |

**Los derechos de acceso solo suman.** Un grupo nuevo no puede quitar un derecho que ya da otro grupo. Para limitar algo heredado hay que usar una regla de registro.

## Tabla de usuarios

| Usuario | Correo de acceso | Puesto | Tipo | Grupos y privilegios por módulo | Justificación |
|---|---|---|---|---|---|
| **Lucía Ferrer** | lucia.ferrer@zumbaruta.example | Comercial | Interno | **Ventas:** Usuario: solo mostrar documentos propios<br>**Contabilidad:** ninguno<br>**Inventario:** ninguno | Prepara sus presupuestos. No necesita ver la cartera de otros ni la contabilidad. |
| **Daniel Radulescu** | daniel.radulescu@zumbaruta.example | Responsable de ventas | Interno | **Ventas:** Administrador<br>**Contabilidad:** ninguno<br>**Inventario:** ninguno | Supervisa y confirma todos los presupuestos. No factura ni cobra. |
| **Inés Aldecoa** | ines.aldecoa@zumbaruta.example | Administración y facturación | Interno | **Ventas:** sin menú<br>**Contabilidad:** Facturación, heredada<br>**Propio:** Zumbaruta / Facturación de pedidos | Factura y cobra. Lee los pedidos, pero no gestiona ventas. |
| **Bruno Talavera** | bruno.talavera@zumbaruta.example | Mozo de almacén | Interno | **Inventario:** Usuario | Valida entregas. No entra en Ventas ni en Facturación. |
| **Noa Ibarra** | noa.ibarra@zumbaruta.example | Comercial de ruta (junior) | Interno | **Ventas:** Usuario: solo mostrar documentos propios, heredado<br>**Propio:** Zumbaruta / Comercial de ruta | Visita fincas y da de alta clientes nuevos, sin borrarlos ni tocar los de otros. |
| **Rocío Benedí** | compras@almendratardia.example | Compras en Almendra Tardía (cliente) | Portal | **Rol / Portal** | Consulta los pedidos y las facturas de su empresa. |
| **Administración de Sistemas** | sistemas@zumbaruta.example | Responsable de sistemas | Interno, rol Administrador | **Rol / Administrador** | Solo configura el sistema. No interviene en la venta. |

Capturas: [lista de usuarios](../capturas/01-usuarios-internos.jpg) y [roles y privilegios de un usuario](../capturas/02-usuario-ines-roles-y-privilegios.jpg).

## Tipos de usuario

| Tipo | Quién es | Acceso | En Zumbaruta |
|---|---|---|---|
| **Interno** | Personal de la empresa | Interfaz de Odoo, según sus grupos | Los 6 usuarios de plantilla |
| **Portal** | Cliente o proveedor | Solo `/my`, con sus propios documentos | Rocío Benedí |
| **Público** | Visitante sin iniciar sesión | Páginas públicas | Sin usuario propio; el registro libre está cerrado |

## Grupos propios

Se configuran en *Ajustes → Usuarios y compañías → Grupos*, con el [modo desarrollador](https://www.odoo.com/documentation/19.0/applications/general/users/access_rights.html) activado.

### Zumbaruta / Comercial de ruta

- **Privilegio:** Zumbaruta: venta en ruta, en la categoría Ventas.
- **Herencia:** Ventas / Usuario: solo mostrar documentos propios → Rol / Usuario.
- **Usuario asignado:** Noa Ibarra.

| Derecho de acceso | Modelo | L | E | C | B |
|---|---|:-:|:-:|:-:|:-:|
| Zumbaruta ruta: clientes | `res.partner` (Contacto) | ✅ | ✅ | ✅ | ❌ |
| Zumbaruta ruta: etiquetas de cliente | `res.partner.category` (Etiquetas del contacto) | ✅ | ✅ | ✅ | ❌ |

| Regla de registro | Modelo | Dominio | Se aplica a |
|---|---|---|---|
| Zumbaruta ruta: solo edita sus clientes | `res.partner` | `['\|', '\|', ('user_id', '=', user.id), ('create_uid', '=', user.id), ('id', '=', user.partner_id.id)]` | Escribir y eliminar |

**Motivo:** el comercial estándar solo puede leer clientes, y quien trabaja en las fincas necesita dar de alta los nuevos. La regla le impide cambiar los de otros compañeros.

Capturas: [derechos](../capturas/03-grupo-propio-permisos-de-acceso.jpg), [herencia](../capturas/04-grupo-propio-heredado.jpg) y [regla](../capturas/05-grupo-propio-regla-de-registro.jpg).

### Zumbaruta / Facturación de pedidos

- **Privilegio:** Zumbaruta: facturar pedidos, en la categoría Contabilidad.
- **Herencia:** Contabilidad / Facturación → Rol / Usuario.
- **Usuario asignado:** Inés Aldecoa.
- **Menú propio:** *Facturación → Clientes → Pedidos por facturar*, visible solo para este grupo.

| Derecho de acceso | Modelo | L | E | C | B |
|---|---|:-:|:-:|:-:|:-:|
| Zumbaruta facturación: asistente de factura | `sale.advance.payment.inv` | ✅ | ✅ | ✅ | ❌ |

| Regla de registro | Modelo | Dominio | Se aplica a |
|---|---|---|---|
| Zumbaruta facturación: pedidos de solo lectura | `sale.order` | `[(0, '=', 1)]` (ningún registro) | Escribir y eliminar |
| Zumbaruta facturación: líneas de pedido de solo lectura | `sale.order.line` | `[(0, '=', 1)]` (ningún registro) | Escribir y eliminar |

**Motivo:** el grupo estándar *Facturación* falla por los dos lados para este puesto:

1. **Le falta un permiso:** no tiene acceso al asistente *Crear factura*, que por defecto es solo de Ventas.
2. **Le sobra otro:** puede escribir en pedidos, así que podría editarlos o cancelarlos.

El derecho de acceso resuelve lo primero y las reglas lo segundo. Como la creación de la factura la hace el propio asistente de Odoo, **sigue funcionando con los pedidos en solo lectura**: se comprobó en pantalla con `S00002`.

Capturas: [menú propio](../capturas/09-facturacion-menu-pedidos-por-facturar.jpg) y [asistente](../capturas/10-facturacion-asistente-crear-factura.jpg).

## Demostración de permisos

Se hicieron **61 comprobaciones con la sesión de cada usuario y las 61 dieron el resultado esperado**. Los flujos principales se repitieron además en la interfaz web.

| Usuario | Aplicaciones que ve | Puede | No puede |
|---|---|---|---|
| **Lucía** (Comercial) | Ventas, Contactos, Conversaciones, Tableros, Aplicaciones | Crear presupuestos; ver solo los suyos | Ver `S00002`; crear, editar o borrar clientes; ver pagos; crear o modificar facturas; crear productos; cambiar grupos; instalar módulos; entrar en Facturación, Inventario o Ajustes |
| **Daniel** (Responsable) | Ventas, Contactos, Conversaciones, Tableros, Aplicaciones | Ver todos los presupuestos; confirmarlos | Borrar clientes; ver pagos; registrar cobros; modificar facturas; entrar en Facturación |
| **Inés** (Administración) | Facturación, Contactos, Conversaciones, Tableros, Aplicaciones | Leer pedidos; usar *Crear factura*; publicar facturas; registrar cobros | Crear, editar, cancelar o borrar pedidos; cambiar líneas; modificar diarios contables; entrar en Ventas |
| **Bruno** (Almacén) | Inventario, Contactos, Conversaciones, Tableros, Aplicaciones | Consultar y validar entregas | Crear presupuestos; ver facturas; usar *Crear factura*; modificar el almacén; entrar en Ventas o Facturación |
| **Noa** (Comercial de ruta) | Ventas, Contactos, Conversaciones, Tableros, Aplicaciones | Crear clientes y etiquetas; editar los clientes que ha creado | Editar clientes de otros; borrar clientes o etiquetas; ver `S00001`, que es de Lucía |
| **Rocío** (Portal) | Portal `/my` | Ver `S00001` e `INV/2026/00002` | Ver documentos de otros clientes; modificar clientes ajenos; crear pedidos; entrar en la interfaz interna, que la devuelve a `/my` |

El menú *Aplicaciones* aparece para todos los usuarios internos, pero Odoo responde `Access Denied` si alguien sin rol de administrador intenta instalar un módulo.

Ejemplo de respuesta de Odoo cuando se abre `S00002` con la sesión de Lucía:

> Lo sentimos, Lucía Ferrer (id=5) no tiene permiso de acceso 'leer' a: Pedido de venta…

Capturas: [Lucía ve solo sus presupuestos](../capturas/06-comercial-solo-sus-presupuestos.jpg), [sus aplicaciones](../capturas/07-comercial-aplicaciones-visibles.jpg) y [portal de Rocío](../capturas/13-portal-cliente-facturas.jpg).

## Límites conocidos

Son comportamientos estándar de Odoo que este diseño no cambia:

- **Ventas puede usar *Crear factura*.** El botón sigue visible para Comercial y Responsable, porque el asistente es de Ventas por defecto. El reparto de tareas asigna esa función a Administración.
- **Inventario / Usuario tiene permiso de escritura en pedidos.** Bruno no tiene menú de Ventas, pero el permiso existe a nivel de modelo.
- **Un comercial puede ver la lista de contactos completa.** La regla de sus documentos se aplica a presupuestos, no a contactos.
