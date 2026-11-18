# ADR 0012 · Sin instancia caliente

- **Estado:** aceptada
- **Fecha:** 2026-11-18
- **Sustituye en parte a:** [ADR 0011](0011-arranque-rapido-en-funciones.md) (la instancia caliente)

## Contexto

El ADR 0011 mantenía una instancia caliente: un Cloudflare Worker de `vfernandezg-infra` llamaba cada 4 minutos a
`/actuator/health/liveness`, y Vercel apaga la instancia tras 5 minutos sin tráfico. Partía de que una instancia
pausada entre peticiones no consume nada facturable.

Eso vale para la CPU (Active CPU), pero no para la memoria. Vercel factura la memoria de la instancia durante toda su
vida, también en reposo (Fluid Provisioned Memory). En el plan Hobby cada instancia tiene 2 GB y la cuenta incluye
360 GB-h al mes, compartidas por todos sus proyectos. Mantener viva una instancia todo el mes cuesta 2 GB × 720 h =
1.440 GB-h, cuatro veces la cuota. Al pasarse, Vercel pausa todos los proyectos de la cuenta.

El 17 de noviembre, cuatro días después de activar el Worker, la cuenta llevaba ya el 90 % de la cuota.

## Decisión

- **Sin instancia caliente.** Se elimina el Worker. La instancia se apaga tras 5 minutos sin tráfico y la primera
  petición después espera el arranque.
- **La app web espera el arranque al recuperar la sesión.** El tope de `AuthStore.restore()` sube de 8 a 20 s
  cuando el navegador tenía sesión, para cubrir el arranque en frío (7-8 s, hasta 15 s) y el despertar de Neon. Con
  8 s, quien volvía tras un rato sin uso acababa en el login con la sesión aún válida.
- Se mantiene el resto del ADR 0011: la caché AOT, el recolector Serial con C1 y la base de datos que puede
  suspenderse.

## Consecuencias

- La primera visita tras 5 minutos sin uso tarda unos 8-10 s (arranque y Neon) y responde. Las siguientes van a la
  velocidad normal mientras haya tráfico.
- El consumo de memoria en Vercel queda en proporción al uso real, dentro del plan gratuito.
- Las tareas programadas (`@Scheduled`) solo avanzan mientras alguien usa la aplicación. Ya eran de mejor esfuerzo
  (ADR 0011): el acceso de pago se compara con la fecha actual en cada petición.

## Alternativas descartadas

- **Ping solo en horario diurno:** 14 h al día siguen siendo 2 GB × 14 h × 30 = 840 GB-h al mes, más del doble de la
  cuota.
- **Plan Pro de Vercel o un servidor siempre encendido:** cuestan dinero para un proyecto de portafolio.
- **Imagen nativa (GraalVM):** bajaría el arranque a décimas de segundo, con los costes que ya recoge el ADR 0011. Es
  la opción si la espera llega a molestar.
