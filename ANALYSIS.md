# Revisión del script de carga

A continuación se detallan los problemas encontrados en el script proporcionado y las acciones recomendadas para resolverlos.

## 1. Bloques duplicados para dimensiones, medidas y propiedades
Entre las líneas 9 y 46 del script aparece dos veces el mismo bloque que carga `DimensionCiclicasVentas`, `MedidasCiclicasVentas`, `ClientesPropiedades`, `ProductosPropiedades` y el `STORE` de `Empresas`. Esta duplicación provoca recargas innecesarias y la segunda ejecución del `STORE` lanza un error si la tabla `Empresas` ya ha sido eliminada o nunca llegó a cargarse.

**Corrección:** eliminar el bloque duplicado y mantener una sola carga de esas tablas auxiliares.

## 2. Uso incoherente de las claves hash
En la tabla `Ventas` se crean claves sintéticas con `AutoNumberHash128('alm',1,CodAlmacen)`, `AutoNumberHash128('prd',1,CodProducto)` y `AutoNumberHash128('agv',1,CodAgenteVenta)`. Sin embargo, en `MovimientosProducto` (tabla de stock) las mismas entidades se generan con `AutoNumberHash128('alm',idEmpresa,CodAlmacen)` y `AutoNumberHash128('prd',idEmpresa,CodProducto)`. Al incorporar el `idEmpresa` en un lado pero no en el otro, los hash nunca coinciden y las tablas quedan desconectadas, lo que explica los datos de stock erróneos.

**Corrección:** utilizar exactamente la misma expresión de hash en todas las tablas que deban asociarse (por ejemplo, incluir `idEmpresa` en ambas o en ninguna).

## 3. Tabla `tabla_clientes_comercial` depende de `Clientes`
El `Resident Clientes` utilizado para construir `tabla_clientes_comercial` requiere que la tabla `Clientes` esté cargada previamente. Si en el nuevo modelo sólo se carga el QVD de Ventas y Stock sin traer `Clientes`, la recarga fallará o generará una tabla vacía.

**Corrección:** asegurar que la tabla `Clientes` se cargue antes de esta sección o sustituir la referencia por el QVD correspondiente.

## 4. Clave sintética por combinación de campos sin llave compuesta
El QVD de `MovimientosProducto` comparte con `Ventas` los campos `idEmpresa`, `FechaSK`, `_idAlmacen` y `_idProducto`. Al no existir un campo llave definido explícitamente para la relación, Qlik crea una tabla sintética cuando los nombres coinciden. La creación inconsistente de hashes mencionada en el punto 2 agrava este problema porque genera múltiples campos comunes que no coinciden.

**Corrección:** unificar las claves hash y, si es necesario, crear una clave compuesta (por ejemplo, `KeyEmpresaProducto = AutoNumberHash128(idEmpresa & '|' & _idProducto)`), y eliminar de la tabla los campos sobrantes para evitar la creación de claves sintéticas.

## 5. Falta de control sobre `WHERE EXISTS(idEmpresa)`
La carga de `MovimientosProducto` filtra con `WHERE EXISTS(idEmpresa)` pero previamente se ha hecho `DROP TABLE tmp_config`, y si la tabla `Empresas` no está cargada cuando se ejecuta esta sección, la condición puede dejar la tabla vacía. Además, se excluye explícitamente `idEmpresa = 3` en ventas pero no en stock, lo que genera discrepancias.

**Corrección:** comprobar que el campo `idEmpresa` exista en el modelo en el momento de la carga o sustituir el `WHERE EXISTS` por un filtro explícito; aplicar el mismo filtro de empresas al stock que a ventas para mantener la coherencia.

## 6. Nombres de campos distintos para la misma entidad
En Ventas se utiliza `CodResponsableVenta`, mientras que en objetivos se carga `CodVendedor` como `[Objetivos.CodigoAgente]`. Al final se hace `DROP FIELD [Objetivos.CodigoAgente], CodigoAgente`, lo que elimina el campo recién calculado y puede romper las asociaciones con los objetivos.

**Corrección:** revisar la nomenclatura y mantener un único campo clave para enlazar ventas, objetivos y permisos.

---

Aplicando estas correcciones se elimina la clave sintética y se restablecen las asociaciones correctas entre ventas y stock en el modelo Qlik Sense.
