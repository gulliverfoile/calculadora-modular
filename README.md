FÍSICA MODULAR — qué es y cómo funciona
========================================================================
Autor:      (tú)
Versión:    1.0
Fecha:      2026-10-05
Archivo:    fisica-modular.html  (autocontenido, se abre con doble clic)

------------------------------------------------------------------------
1. QUÉ ES
------------------------------------------------------------------------
Una calculadora de física que corre en el navegador sin instalar nada.
Resuelve el mismo problema de "transportar flujo por una red con
pérdidas" en tres dominios distintos:

  - Fluido      → aire o agua por conductos (troncal + N ramales)
  - Eléctrico   → caída de tensión en línea con N cargas en paralelo
  - Térmico     → conducción de calor por capas en serie

Los tres comparten la misma matemática subyacente (la analogía entre
potencial, flujo y resistencia), así que se implementan con la misma
arquitectura: un dominio es un objeto con un schema y una función
solve() pura. La UI se autogenera a partir del schema.

No hay dependencias externas. No hay build step. Un solo archivo HTML.

------------------------------------------------------------------------
2. POR QUÉ EXISTE
------------------------------------------------------------------------
- Para comparar diseños de canalización (un tubo largo vs varios cortos)
  con números, no con intuiciones.
- Para tener un banco de pruebas de arquitectura hexagonal en frontend
  puro, sin frameworks.
- Para enseñar cómo la misma ecuación aparece en dominios distintos:
    ΔV = I·R         (eléctrico, ley de Ohm)
    Δp = R_h·Q       (fluido, Hagen-Poiseuille)
    ΔT = R_th·Q̇      (térmico, ley de Fourier)
  Son la misma cosa con nombres cambiados.

------------------------------------------------------------------------
3. ARQUITECTURA
------------------------------------------------------------------------
Patrón: hexagonal (puertos y adaptadores).

    +--------------------------------------------------+
    |                    UI (adaptador)                |
    |  - Genera formularios desde el schema            |
    |  - Pinta métricas y gráfica                      |
    |  - Emite eventos al bus                          |
    +-----------------------+--------------------------+
                            |
                    EventBus (puerto)
                            |
    +-----------------------v--------------------------+
    |                  Dominios (núcleo)               |
    |  - Física pura, sin DOM, sin eventos             |
    |  - solve(input) → { metrics, curves }            |
    +--------------------------------------------------+

La regla: los dominios NO saben que existe el DOM.
La UI NO sabe que existen dominios concretos, solo consume schemas.
Todo se conecta por eventos a través del bus.

------------------------------------------------------------------------
4. MÓDULOS DEL CÓDIGO
------------------------------------------------------------------------
Todos viven bajo el namespace global "App" para no ensuciar el scope.

App.EventBus (Módulo 1)
    Puerto de comunicación. Métodos:
      on(evento, handler)      → devuelve función para desuscribir
      off(evento, handler)
      emit(evento, payload)
    Los módulos no se llaman entre sí: emiten y escuchan.
    Esto permite añadir logging, persistencia o telemetría sin tocar
    la lógica de negocio.

App.Physics (Módulo 2)
    Funciones puras, sin estado, reutilizables:
      airProps(T)         → densidad, viscosidad, conductividad, cp del aire
      waterProps()        → constantes del agua en rango 10-80°C
      frictionFactor(Re, D)  → Darcy (64/Re laminar, Haaland turbulento)
      hInternal(Re, Pr, k, D) → convección interna (Dittus-Boelter)
      overallU(hi, D, tIns)  → coeficiente global de un tubo aislado
      ntuEffectiveness(NTU)  → ε = 1 - e^(-NTU)
    Son testeables en aislamiento, sin navegador.

App.Registry (Módulo 3)
    Diccionario de dominios. Métodos:
      register(domain)   → valida y guarda
      get(id)            → recupera
      list()             → array de todos
    Un dominio es un objeto con:
      - id, name           (identificación para la UI)
      - description        (texto informativo)
      - schema             (array de campos de entrada)
      - solve(input)       (función pura, devuelve { metrics, curves })

Dominios registrados (uno por bloque)
    fluid     Aire o agua por troncal + N ramales. Resuelve pérdida de
              carga con Darcy-Weisbach y caída térmica con NTU.
              Métricas: velocidad, Re, Δp, T de llegada, potencia útil.

    electric  Caída de tensión en línea con N cargas. Usa R = ρL/A,
              ΔV = I·R, pérdida Joule = I²R.
              Métricas: I, ΔV, ΔV%, P disipada, P entregada, J.

    thermal   Conducción por capas en serie. Usa R_th = d/(kA),
              ΔT = Q̇·R_th.
              Métricas: R total, ΔT, T interfaz, flujo por área.

App.Test (Módulo 4)
    Micro-framework de tests. Métodos:
      it(suite, name, fn)      → registra y ejecuta un test
      assert.close(a, b, tol)  → comparación numérica
      assert.equal(a, b)       → comparación exacta
      assert.ok(cond)          → booleano
      run()                    → devuelve resultados

Tests (Módulo 5)
    Cubren física pura, sin tocar el DOM:
      - Propiedades del aire a 20°C ≈ valores estándar
      - Factor de fricción laminar = 64/Re
      - NTU monótona creciente, límites en 0 y ∞
      - Fluid: caudal 0 ⇒ velocidad 0
      - Fluid: más longitud ⇒ más pérdida de carga
      - Fluid: más longitud ⇒ menos T de llegada
      - Electric: V=230, P=2300 ⇒ I=10 A
      - Electric: conservación de energía (P_ent = P_total - P_dis)
      - Electric: aluminio pierde más que cobre
      - Thermal: ΔT = Q̇ · R_th
    Cada test lleva un comentario explicando QUÉ comprueba y POR QUÉ.

App.UI (Módulo 6)
    Adaptador. Responsabilidades:
      - Pintar pestañas de dominios
      - Generar formularios a partir del schema
      - Suscribirse a "ui.domain.selected" y "ui.input.changed"
      - Resolver el dominio activo y pintar resultados
      - Dibujar la gráfica en canvas
      - Renderizar el panel de tests

Bootstrap (Módulo 7)
    En DOMContentLoaded, llama a App.UI.init().

------------------------------------------------------------------------
5. FLUJO DE DATOS
------------------------------------------------------------------------
Usuario cambia un input
        │
        v
UI emite "ui.input.changed" { domainId, key, value }
        │
        v
EventBus despacha a los suscriptores
        │
        v
UI actualiza el estado local y llama a renderDomain()
        │
        v
renderDomain pide domain.solve(state)
        │
        v
solve() calcula (física pura) y devuelve { metrics, curves }
        │
        v
UI pinta métricas en tarjetas y curvas en canvas

En ningún punto el dominio sabe que existe la UI.

------------------------------------------------------------------------
6. CÓMO AÑADIR UN DOMINIO NUEVO
------------------------------------------------------------------------
Al final del bloque de dominios, añade:

    App.Registry.register({
      id: 'acoustic',
      name: '🔊 Acústico',
      description: 'Frecuencia fundamental de un tubo abierto.',
      schema: [
        { id:'L', label:'Longitud', unit:'m', default:1, type:'number' },
      ],
      solve(input) {
        const c = 343;                      // velocidad del sonido en aire
        const f = c / (2 * input.L);
        return {
          metrics: [
            { label:'Frecuencia', value: f.toFixed(1), unit:'Hz' },
          ],
          curves: [],
        };
      }
    });

Recarga la página y aparece como pestaña nueva. La UI se autogenera.
Los tests lo ignoran si no le añades ninguno.

------------------------------------------------------------------------
7. CÓMO EXTRAERLO A ARCHIVOS
------------------------------------------------------------------------
Cada bloque "App.X = ..." va a su propio archivo .js:
    event-bus.js
    physics.js
    registry.js
    domains/fluid.js
    domains/electric.js
    domains/thermal.js
    test-framework.js
    tests.js
    ui.js
    main.js

En el HTML:
    <script type="module" src="main.js"></script>

main.js importa todo lo demás. No hace falta bundler.

------------------------------------------------------------------------
8. CÓMO ADAPTARLO A CASOS NUEVOS
------------------------------------------------------------------------
Es un ejemplo de arquitectura, no un producto cerrado.
Cambios naturales:

  - Persistir estado en localStorage:
      Escuchar "ui.input.changed" y guardar. Al arrancar, precargar.

  - Leer parámetros de la URL:
      Al iniciar, leer ?domain=X&param=Y y emitir los eventos.

  - Añadir dominio con gráfica especial:
      solve() puede devolver curves con cualquier forma.
      La UI ya pinta cualquier array de puntos { x, y }.

  - Añadir tests de integración dominio + EventBus:
      Suscribirse a un evento y comprobar que se emite.

  - Cambiar la UI por otra (React, Vue, canvas puro):
      Solo hay que reescribir App.UI. Los dominios y tests no cambian.

------------------------------------------------------------------------
9. VERIFICACIÓN
------------------------------------------------------------------------
Al abrir el archivo, la sección "Tests" al final muestra el resultado.
Debe aparecer:
    ✓ N pasan   (N = número total de tests, actualmente ~20)
    (0 tests)

Si algún test falla, se muestra en rojo con el motivo.
Los tests son la especificación viva de la física del sistema.

Consola del navegador (F12) al arrancar:
    [Física modular] Sistema arrancado.
    Dominios registrados: fluid, electric, thermal
    Para añadir un dominio nuevo: App.Registry.register({...})

------------------------------------------------------------------------
10. LIMITACIONES CONOCIDAS
------------------------------------------------------------------------
- Los dominios son deliberadamente simples. Cada uno modela un caso
  concreto, no pretende ser una herramienta de ingeniería de precisión.

- Fluid: el modelo asume troncal único + N ramales paralelos. No
  soporta ramificaciones en árbol a más de un nivel.

- Electric: cálculo resistivo puro a 20°C. No tiene en cuenta
  temperatura del conductor, factor de potencia, ni inductancia.

- Thermal: conducción estacionaria 1D. No modela radiación ni
  convección externa.

- No hay validación de entradas. Si metes valores absurdos (negativos,
  cero) el resultado puede ser NaN o Infinity. Los tests cubren el
  rango razonable, no el patológico.

- Todo el estado vive en memoria. Refrescar la página pierde los
  valores introducidos.

------------------------------------------------------------------------
11. REFERENCIAS
------------------------------------------------------------------------
La analogía entre dominios es un tema clásico:
  - Ley de Ohm ↔ Hagen-Poiseuille ↔ Ley de Fourier
  - Modelado "lumped element" en ingeniería
  - Teoría constructal (Bejan) para redes de distribución
  - Ley de Murray para ramificaciones óptimas

Las fórmulas usadas:
  - Darcy-Weisbach:  Δp = f · (L/D) · ρv²/2
  - Colebrook/Haaland: factor de fricción turbulento
  - Dittus-Boelter:  Nu = 0.023 · Re^0.8 · Pr^0.4
  - NTU-ε:           ε = 1 - e^(-NTU)
  - Ley de Ohm:      ΔV = I·R
  - Resistencias en serie y paralelo
  - R_th = d / (k·A)

========================================================================
FIN DEL DOCUMENTO
