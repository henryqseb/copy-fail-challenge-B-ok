# Reporte Técnico — CVE-2026-31431 "Copy Fail"

## Introducción

Durante este laboratorio se analizó la vulnerabilidad CVE-2026-31431,
conocida como "Copy Fail", presente en el subsistema criptográfico
del kernel Linux.

El objetivo fue:
- comprobar la vulnerabilidad,
- explotar el fallo,
- aplicar una mitigación temporal,
- implementar un parche permanente,
- y verificar que el exploit dejara de funcionar.

---

## Descripción de la vulnerabilidad

La vulnerabilidad se encuentra en:

  crypto/algif_aead.c

y afecta a la interfaz AF_ALG del kernel Linux.

El problema ocurre por un manejo incorrecto de los buffers
scatter-gather utilizados durante operaciones AEAD.

La versión vulnerable reutilizaba el mismo buffer para
entrada y salida, generando una condición peligrosa que
permitía modificar el page cache del kernel.

Como consecuencia, un usuario sin privilegios podía alterar
temporalmente binarios SUID cargados en memoria y obtener
privilegios root.

---

## Funcionamiento del exploit

El exploit público utilizado fue un script Python pequeño
que aprovecha:

- sockets AF_ALG,
- operaciones AEAD,
- llamadas splice(),
- y el módulo authencesn.

El ataque modifica páginas de memoria asociadas a:

  /usr/bin/su

Cuando el binario se ejecuta, el kernel utiliza la versión
alterada presente en memoria y permite escalar privilegios.

En la práctica, el exploit logró cambiar el usuario:

  student -> root

confirmado mediante:

  id
  whoami

---

## Mitigación temporal

Como mitigación rápida se bloqueó el módulo vulnerable:

  rmmod algif_aead

y luego:

  echo "install algif_aead /bin/false" > /etc/modprobe.d/block-algif.conf

Esto evita que el módulo pueda cargarse nuevamente.

La desventaja es que ciertos programas que dependen de
AF_ALG podrían dejar de funcionar correctamente.

---

## Parche permanente

El parche se aplicó directamente sobre:

  crypto/algif_aead.c

La modificación principal consistió en separar correctamente
los buffers de origen y destino dentro de:

  aead_request_set_crypt()

Código vulnerable:

  areq->first_rsgl.sgl.sgt.sgl

Código corregido:

  rsgl_src

Con esto se evita la escritura insegura sobre el page cache
del kernel.

---

## Resultados obtenidos

Después de recompilar el kernel parcheado:

- el exploit dejó de funcionar,
- no se obtuvo acceso root,
- el usuario permaneció como student.

Además, el exploit mostró el mensaje:

  su: Module is unknown

confirmando que la vulnerabilidad quedó mitigada correctamente.

---

## Conclusión

La práctica permitió comprender cómo un error lógico dentro
del kernel Linux puede convertirse en una vulnerabilidad
crítica de escalamiento de privilegios.

También permitió observar la diferencia entre:
- una mitigación temporal,
- y una corrección permanente mediante parcheo del kernel.

