<div align="center">

# gross-back

### Las fees son ingreso bruto: mitad a crecimiento, mitad a los holders

![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black) ![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white) ![Express](https://img.shields.io/badge/Express-000000?style=for-the-badge&logo=express&logoColor=white) ![WebSocket](https://img.shields.io/badge/WebSocket-1F2937?style=for-the-badge&logo=socketdotio&logoColor=white) ![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=for-the-badge&logo=mongodb&logoColor=white) ![Ethers.js](https://img.shields.io/badge/Ethers.js-2535A0?style=for-the-badge&logo=ethereum&logoColor=white)

[**📦 Repositorio**](https://github.com/Harkor421/gross-back)

</div>

---

## 📖 Sobre el proyecto

Backend de **$GROSS**, el repartidor. Las creator fees son el ingreso bruto del token y se parten: un porcentaje configurable (50% por defecto) va al wallet de crecimiento (boosts de DEX y marketing) y el resto paga $1 de ETH nativo a cada holder, del bolsillo más grande al más pequeño, cada 5 minutos.

## ✨ Qué hace

- Split configurable entre wallet de crecimiento y holders
- Transferencias nativas: sin swap en DEX, nada que pueda revertir
- Recorrido de holders de mayor a menor hasta agotar el pot
- Sin reclamos: el pago llega solo
- Estado y pagos en vivo por WebSocket

## 🧰 Stack

| | |
|---|---|
| **Lenguajes y runtime** | ![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black) ![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white) |
| **Backend** | ![Express](https://img.shields.io/badge/Express-000000?style=for-the-badge&logo=express&logoColor=white) ![WebSocket](https://img.shields.io/badge/WebSocket-1F2937?style=for-the-badge&logo=socketdotio&logoColor=white) |
| **Datos** | ![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=for-the-badge&logo=mongodb&logoColor=white) |
| **Web3** | ![Ethers.js](https://img.shields.io/badge/Ethers.js-2535A0?style=for-the-badge&logo=ethereum&logoColor=white) |

## 🚀 Empezar

```bash
git clone https://github.com/Harkor421/gross-back.git
cd gross-back
npm install
npm run start
```

## 📜 Scripts

| Comando | Qué hace |
|---|---|
| `npm run start` | Arranca la aplicación |

---

<div align="center">

Hecho por [**Samir González**](https://github.com/Harkor421)

</div>
