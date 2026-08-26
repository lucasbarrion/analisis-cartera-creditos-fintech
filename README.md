
# Análisis de Cartera de Créditos — Fintech de Préstamos Personales

## 📌 Contexto y Problema de Negocio

Una fintech argentina dedicada al otorgamiento de préstamos personales necesita entender el comportamiento de su cartera de créditos para tomar decisiones de negocio más informadas.
La compañía cuenta con dos sistemas operativos que generan datos constantemente — originación de créditos y cobranzas/pagos — pero esa información vive dispersa y sin un proceso analítico que la transforme en indicadores accionables.

El objetivo de este proyecto fue diseñar y construir un Data Warehouse en la nube, capaz de recibir datos nuevos de forma continua, transformarlos, y ponerlos a disposición de un dashboard de Business Intelligence que responda las siguientes preguntas de negocio:

1. **¿Cómo se comporta la mora según el segmento de riesgo del cliente?**
2. **¿Cómo evoluciona la mora a medida que maduran los créditos (análisis de cohortes / vintage)?**
3. **¿Cuál es la rentabilidad real de la cartera una vez ajustada por inflación?**

## 🎯 Objetivo del Proyecto

El objetivo de este proyecto es ayudar a la fintech a ordenar y centralizar la información que generan sus sistemas de originación de créditos y cobranzas en un Data Warehouse propio,
capaz de mantenerse actualizado de forma automática a medida que se generan nuevos créditos y pagos.

Esto le permite a la empresa contar con un modelo de datos consistente y confiable a lo largo del tiempo, sobre el cual pueda apoyarse para el análisis de su cartera de créditos — 
sin depender de reportes manuales ni de procesos que haya que rehacer cada vez que se necesite una nueva mirada del negocio.

A partir de ese Data Warehouse, el proyecto construye además un tablero de Business Intelligence que responde de forma visual e interactiva a las preguntas de negocio definidas junto a la empresa:
cómo se comporta la mora según el segmento de riesgo del cliente, cómo evoluciona la mora a medida que maduran los créditos, y cuál es la rentabilidad real de la cartera una vez ajustada por inflación.
