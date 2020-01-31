---
title:  "Introducción a ISCSI"
excerpt: "Instalación y configuración de un servidor y clientes ISCSI"
date:   2020-02-03 10:59:00
categories: [Sistemas]
tags: [ISCSI,filesystem]
---

## Introducción

**ISCSI** es un estandar del protocolo **SCSI** que permite la distribución de dispositivos de bloques a través de **TCP/IP**. Dicho protocolo se utiliza por norma general en escenarios donde se ve involucrada una [SAN](https://es.wikipedia.org/wiki/Red_de_%C3%A1rea_de_almacenamiento) o al menos una [NAS](https://es.wikipedia.org/wiki/Almacenamiento_conectado_en_red), que se encargan de proveer almacenamiento en la red.
Gracias a este estandar podemos tener centralizado el almacenamiento de múltiples máquinas y hasta podríamos hacer que estas arrancasen con un dispositivo de bloques proporcionado por **ISCSI**.
Para ver su funcionamiento, vamos a plantear un escenario donde tendremos un servidor (_debian_) y dos clientes (_debian_ y _windows_). Si queréis replicar el escenario, os dejo el [Vagrantfile](/docs/EscenarioISCSI/Vagrantfile) que he utilizado. Intentaremos conseguir los siguientes objetivos:

* Crea un target con una LUN y conéctala a un cliente GNU/Linux. Explica cómo escaneas desde el cliente buscando los targets disponibles y utiliza la unidad lógica proporcionada, formateándola si es necesario y montándola.
* Utiliza systemd mount para que el target se monte automáticamente al arrancar el cliente.
* Crea un target con 2 LUN y autenticación por CHAP y conéctala a un cliente windows. Explica cómo se escanea la red en windows y cómo se utilizan las unidades nuevas (formateándolas con NTFS).

_Para más información sobre ISCSI consulta_ [aquí](https://es.wikipedia.org/wiki/ISCSI).



## Configuración del servidor

Antes de configurar **ISCSI** en el servidor, vamos a crear un _RAIDZ-2_, tal y como hicimos en la entrada de [ZFS](/Sistema-de-ficheros-ZFS/)
