# Vega-Lite: Gráficos Interativos

```js
import * as aq from 'npm:arquero';
import { op, table } from 'npm:arquero';
import * as vega from "npm:vega";
import * as vegaLite from "npm:vega-lite";
import * as vegaLiteApi from "npm:vega-lite-api";
import * as vegaTooltip from "npm:vega-tooltip";
```

```js
const vl = vegaLiteApi.register(vega, vegaLite, {
  init: (view) => {
    view.tooltip(new vegaTooltip.Handler().call);
    if (view.container()) view.container().style["overflow-x"] = "auto";
  }
}); // Configuring the Vega-Lite tooltip plugin 
```

```js
const energyConsumptionData = FileAttachment("data/br_mme_consumo_energia_eletrica_uf.csv").csv();
const brazilStateCodes = FileAttachment("data/br_bd_diretorios_brasil_uf.csv").csv();
```

```js
let brazilStateCodesAq = aq
    .from(brazilStateCodes)
```

```js
let energyConsumptionDataAq = aq
    .from(energyConsumptionData)
    .join_left(brazilStateCodesAq, ['sigla_uf', 'sigla'])
    .select(aq.not('regiao', 'sigla')) // Removing columns
```

```js
let totalConsumptionByState = energyConsumptionDataAq
    .filter((d) => d.ano == "2023" && d.tipo_consumo != "Cativo")
    .groupby("sigla_uf", "id_uf", "nome")
    .rollup({ consumo_total: aq.op.sum("consumo") }).objects()
```

```js
let brazilMap = await vl.render({
    spec:{
        width: 550,
        height: 400,
        autosize: {type: "fit", contains: "padding"},
        title: {
            text: "Consumo total de energia por estado em 2023",
            anchor: "start",
            offset: 20,
            fontSize: 20,
            subtitle: [`Clique em um estado para filtrar o gráfico ao lado`, `Clique novamente para voltar à agregação nacional.`],
            subtitleFontSize: 16
            },
        data: {
            url: "https://servicodados.ibge.gov.br/api/v3/malhas/paises/BR?formato=application/json&qualidade=intermediaria&intrarregiao=UF",
            format: {
            type: "topojson",
            feature: "BRUF"
            }
        },
        projection: {
            type: "mercator",
        },
        transform: [
            {
                lookup: "properties.codarea", // Vinculates 'properties.codarea' from TopoJSON
                from: {
                data: { values: totalConsumptionByState }, // Energy consumption data
                key: "id_uf", // Field to link with TopoJSON
                fields: ["consumo_total", "nome"] // Data to be used on the map
                }
            },
            {
                calculate: "datum.consumo_total || 0", // Defines 0 for dataless states
                as: "consumo_total"
            }
            ],
        selection: {
            stateSelector: {
                type: "point",
                fields: ["properties.codarea", "nome"], // Field used to give the signal to the Listener
            }
        },
        mark: {
            type: "geoshape",
            stroke: "white",
            strokeWidth: 0.5
            },
        encoding: {
            color: {
                field: "consumo_total",
                type: "quantitative",
                scale: {
                scheme: "blues" 
                },
                title: "Consumo em MWh",
                legend: {
                    direction: "vertical",
                    orient: "bottom-left",
                    offset: 10
                }
            },
            tooltip: [
                { field: "nome", type: "nominal", title: "Estado:" },
                { field: "consumo_total", type: "quantitative", title: "Consumo Total (MWh):" }
                ]
            }
        }
})
```

```js
let clickedStateCode = Mutable('0');
let setClickedStateCode = (x) => {clickedStateCode.value = x};

let clickedStateName = Mutable('Brasil');
let setClickedStateName = (x) => {clickedStateName.value = x};
```

```js
function stateSelectorHandler(name, value) {

    const newSelection = value["properties\\.codarea"]?.[0] ?? '0';
    const stateSelected = value.nome?.[0] ?? 'Brasil'
    
    if (clickedStateCode === newSelection) {
        setClickedStateCode('0');
        setClickedStateName('Brasil');
    } else {
        setClickedStateCode(newSelection);
        setClickedStateName(stateSelected);
    }
}

brazilMap.value.addSignalListener("stateSelector", stateSelectorHandler);

invalidation.then(() => {
    brazilMap.value.removeSignalListener("stateSelector", stateSelectorHandler);
});
```

```js
let brazilTotalConsumption = energyConsumptionDataAq
    .filter((d) => d.ano == "2023" && d.tipo_consumo != "Cativo" && d.tipo_consumo != "Total")
    .groupby("tipo_consumo")
    .rollup({
        consumo_ano: aq.op.sum("consumo")
        })
    .derive({
        sigla_uf: () => "BR",
        id_uf: () => "0",
        nome: () => "Brasil"
        })
    .select(['sigla_uf', 'id_uf', 'nome', 'tipo_consumo', 'consumo_ano'])
```

```js
let consumptionByState = energyConsumptionDataAq
    .filter((d) => d.ano == "2023" && d.tipo_consumo != "Cativo" && d.tipo_consumo != "Total") 
    .groupby("sigla_uf", "id_uf", "nome", "tipo_consumo")
    .rollup({ consumo_ano: aq.op.sum("consumo") })
```

```js
let consumptionTreatedData = consumptionByState
    .concat(brazilTotalConsumption)
    .filter(aq.escape((d) => d.id_uf == clickedStateCode))
```

```js
let barChart = await vl.render({
    spec: {
        width: 500,
        height: 400,
        autosize: {type: "fit", contains: "padding"},
        title: {
            text: `Consumo de energia por categoria: ${clickedStateName}`,
            anchor: "start",
            offset: 20,
            fontSize: 20,
            subtitle: [`Interaja com o mapa para mudar o estado`],
            subtitleFontSize: 16
            },
        data: { values: consumptionTreatedData },
        mark: { type: "bar" },
        encoding: {
            x: {
                field: "consumo_ano", 
                type: "quantitative",
                axis: { grid: true },
                title: "Consumo Anual (MWh)"
                },
            y: {
                field: "tipo_consumo",
                type: "nominal",
                axis: { grid: false },
                title: "Tipo de Consumo"
                }
        }
  }
});
```

<style>
.content {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 50px;
    max-width: 1200px;
    margin: 0;
}
.chart {
    max-width: 550px;
}

@media (max-width: 768px) {
    .content {
        grid-template-columns: 1fr;
    }
}
</style>

<div class="content">
  <div class="chart">${brazilMap}</div>
  <div class="chart">${barChart}</div>
</div>

O código-fonte completo do dashboard está [neste repositório do GitHub](https://github.com/alessandrofajr/vega-lite-interactive-charts). Escrevi um guia [detalhando o processo de desenvolvimento aqui](https://alessandrofajr.com/blog/vega-lite-graficos-interativos-observable-framework/).