# Unidades

Los sistemas de planificación de recursos empresariales (ERP) presentan
distintas formas de manejar las unidades en las que se comercializan los
productos. Se identifican tres enfoques principales en la gestión de unidades:

**Básico**: Este nivel permite el manejo de unidades tanto a nivel de producto como
en las líneas de factura, sin embargo, no admite conversiones entre distintas
unidades. La unidad se utiliza principalmente para propósitos de visualización
en las facturas.

**Intermedio**: En este nivel, el sistema es capaz de gestionar unidades a nivel de
producto, permitiendo la conversión entre unidades de inventario, de venta y de
compra. Las facturas reflejan las cantidades únicamente en las unidades
específicas asignadas al producto.

**Avanzado**: Este enfoque ofrece un manejo completo de las unidades a nivel de
producto, incluyendo la capacidad de convertir entre unidades de inventario, de
venta, de compra, y cualquier otra unidad definida en tablas de conversión. Las
unidades utilizadas en las facturas pueden corresponder a cualquiera de las
definidas en el sistema, siempre y cuando exista la posibilidad de convertirlas
a la unidad del producto.

Nuestra solución integra estos tres modelos de gestión de unidades mediante una
estructura de datos unificada, compuesta por las siguientes tablas:

* `Unit`: Define las unidades gestionadas en el sistema.
* `UnitList`: Enumera las listas de unidades que pueden convertirse entre sí.
* `UnitListDetail`: Detalla los factores de conversión entre unidades.

Estas tablas interactúan con:

* `Product`: Cataloga los productos disponibles para la venta.
* `SaleDocumentDetail`: Registra las líneas de detalle en las facturas de venta.
* `Stock`: Mantiene el control de las existencias de productos en inventario.

Vamos a comenzar con la explicación del modelo **avanzado** para luego ir viendo
como esta versión 'completa' se va reduciendo a las versiones **intermedio** y
**básico**.

Cada producto (`Product`) se asocia a una unidad base (`unit`), que facilita el
manejo de sus cantidades independientemente de las operaciones de compra,
inventario o venta. Aunque generalmente coincide con la unidad de inventario, no
es una obligación. Adicionalmente, se manejan tres unidades específicas para
procesos de compra (`purchaseUnit`), inventario (`inventoryUnit`) y venta
(`saleUnit`), junto a tres parámetros que permiten convertir entre estas unidades
y la unidad base. 

El elemento básico de referencia de unidades es el id de la unidad; sin embargo,
cada vez que se almacena el id de la unidad, también almacenamos el nombre y los
factores de conversión entre la utilizada y la unidad base del producto. De esta
forma, se garantiza la integridad relacional entre las tablas y se evita la
necesidad de realizar un join para obtener el nombre de la unidad.

Estamos usando dos números para hacer la conversión entre unidades `alternative`
y `base`. Si tenemos las unidades en `purchaseUnit` y queremos convertirla en
`unit` usamos la fórmula:

$$
[\text{cantidad en unit}] = 
\frac{[\text{cantidad en purchaseUnit}] \times \text{purchaseUnitBase}}
     {\text{purchaseUnitAlternative}}
$$

Esta forma hace obvio qué hay que multiplicar y qué hay que dividir. Lo típico
es tener relaciones de la forma `1 [docena] = 12 [unidad]`. Dependiendo de la
base tendríamos que multiplicar por `12` o multiplicar por `0.0833333` para
llegar a la otra unidad. Esta forma también permite lidiar con esos decimales
que pueden salir en ciertas conversiones.

Finalmente, cada producto puede tener una lista de unidades que se pueden
utilizar en el sistema para comprar, vender o inventariar. La lista de unidades
tiene una unidad base que debe coincidir con el `unit` del producto y debe
contener, como mínimo, las unidades de compra, inventario y venta. Esta lista se
guarda en `UnitList` y `UnitListDetail`.

Con todo esto en mente, el modelo de datos para el `Product` contiene las
siguientes columnas de definición:

* `unitId` - El id de la unidad base del producto.
* `unitName` - El nombre de la unidad base.
* `unitListId` - El id de la lista de unidades que se pueden convertir entre sí.

Y estas columnas de conversión:

* `purchaseUnitId` - El id de la unidad en la que se compra el producto.
* `purchaseUnitName` - El nombre de la unidad en la que se compra el producto.
* `purchaseUnitAlternative` - La cantidad de unidades de `purchaseUnit`.
* `purchaseUnitBase` - La cantidad de unidades de `unit`.
* `inventoryUnitId` - El id de la unidad en la que se mide el inventario.
* `inventoryUnitName` - El nombre de la unidad en la que se mide el inventario.
* `inventoryUnitAlternative` - La cantidad de unidades de `inventoryUnit`.
* `inventoryUnitBase` - La cantidad de unidades de `unit`.
* `saleUnitId` - El id de la unidad en la que se vende el producto.
* `saleUnitName` - El nombre de la unidad en la que se vende el producto.
* `saleUnitAlternative` - La cantidad de unidades de `saleUnit`.
* `saleUnitBase` - La cantidad de unidades de `unit`.

El principal otro lugar donde se manejan las unidades es en las líneas de
factura almacenadas en `SaleDocumentDetail`. Cada línea de factura tiene una
cantidad en la que se hizo la venta (`displayUnit`) con su correspondiente
precio (`displayPrice`) que está asociado a la unidad de la transacción. Esa
cantidad hay que convertirla a la unidad base del producto (`unit`) para poder
hacer los análisis en una unidad común. Esa unidad común luego se puede traducir
a cualquier otra unidad del sistema para la cual exista una conversión. 

Al momento de hacer cualquier cambio de la unidad también es necesario cambiar
el precio de la línea de factura, tanto para la moneda local como para la moneda
original en la que se hizo la transacción. Con esto se garantiza que el precio
de la línea de factura sea el mismo independientemente de la unidad en la que se
mida la cantidad.

El modelo de datos para `SaleDocumentDetail` contiene las siguientes columnas de
definición:

* `productId` - El id del producto vendido.

La unidad y precio de la transacción:

* `displayQuantity` - La cantidad en la que se hizo la venta.
* `displayUnitId` - El id de la unidad en la que se hizo la venta.
* `displayUnitName` - El nombre de la unidad en la que se hizo la venta.
* `displayUnitAlternative` - La cantidad de unidades de `displayUnit`.
* `displayUnitBase` - La cantidad de unidades de `unit`.
* `displayPrice` - El precio de la venta

Lo que se traduce en la versión normalizada de cantidad, precio y moneda:

* `quantity` - La cantidad en la que se hizo la venta convertida a la unidad base.
* `unitId` - El id de la unidad base del producto.
* `unitName` - El nombre de la unidad base del producto.
* `price` - El precio de la venta convertido a la unidad base.
* `originalCurrency` - La moneda en la que se hizo la venta convertida a la unidad base.
* `originalPrice` - El precio de la venta convertido a la unidad base y en la moneda original.
