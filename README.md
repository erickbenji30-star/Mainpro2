# MAINPRO
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>MAINPRO | Sistema de Mantenimiento Cloud</title>
<link href="https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;500;700&display=swap" rel="stylesheet">

<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.5.29/jspdf.plugin.autotable.min.js"></script>

<script src="https://www.gstatic.com/firebasejs/9.17.1/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.17.1/firebase-database-compat.js"></script>

<style>
    body{ font-family:'Roboto',sans-serif; background:#e9ecef; margin:0; }
    .topbar{ background:#800020; color:white; padding:15px 30px; font-size:20px; font-weight:500; }
    .container{ max-width:1100px; margin:auto; background:white; padding:25px; margin-top:20px; border-radius:8px; box-shadow:0 4px 10px rgba(0,0,0,0.1); }
    .logo{ display:block; margin:auto; width:220px; margin-bottom:25px; }
    h2{ color:#800020; border-bottom:2px solid #800020; padding-bottom:8px; margin-top:30px; }
    table{ width:100%; border-collapse:collapse; margin-bottom:15px; }
    th{ background:#800020; color:white; padding:10px; }
    td{ padding:10px; border-bottom:1px solid #ddd; }
    input,textarea{ width:100%; padding:8px; border:1px solid #ccc; border-radius:4px; font-family:inherit; margin-bottom:10px; }
    button{ background:#800020; color:white; border:none; padding:10px 15px; border-radius:4px; cursor:pointer; margin-top:10px; }
    button:hover{ background:#600018; }
    .btn-secondary{ background:#444; }
</style>
</head>
<body>

<div class="topbar">Sistema de Gestión de Mantenimiento</div>

<div class="container">
    <img class="logo" src="https://i.imgur.com/53mj1cB.png">

    <h2>Registrar Refacciones</h2>
    <table>
        <thead>
            <tr><th>ID</th><th>Refacción</th><th>Cantidad</th><th>Proveedor</th></tr>
        </thead>
        <tbody id="refacciones">
            <tr>
                <td><input class="id"></td>
                <td><input class="nombre"></td>
                <td><input class="cant" type="number"></td>
                <td><input class="prov"></td>
            </tr>
        </tbody>
    </table>
    <button class="btn-secondary" onclick="agregarFila()">Agregar refacción</button>

    <h2>Nueva Orden de Trabajo</h2>
    <label>Equipo</label><input id="equipo">
    <label>Trabajo</label><textarea id="trabajo"></textarea>
    <label>Responsable</label><input id="responsable">
    <button onclick="crearOrden()">Generar Orden</button>

    <h2>Historial de Órdenes</h2>
    <button onclick="exportarExcel()">Exportar a Excel</button>
    <table id="tablaOrdenes">
        <thead>
            <tr>
                <th># Orden</th><th>Equipo</th><th>Trabajo</th><th>Refacciones</th><th>Responsable</th><th>Fecha</th><th>PDF</th>
            </tr>
        </thead>
        <tbody id="historial"></tbody>
    </table>
</div>

<script>
    // 1. CONFIGURACIÓN DE FIREBASE (Datos de tu imagen)
    const firebaseConfig = {
        apiKey: "AIzaSyBrOrTrLuXL-Dp8U_Djqrfuw8bemgdzBUA",
        authDomain: "mainpro4602.firebaseapp.com",
        databaseURL: "https://mainpro4602-default-rtdb.firebaseio.com",
        projectId: "mainpro4602",
        storageBucket: "mainpro4602.firebasestorage.app",
        messagingSenderId: "289547207889",
        appId: "1:289547207889:web:00b913607c5454c5d171f0",
        measurementId: "G-WCH4D87ZZS"
    };

    // Inicializar Firebase
    firebase.initializeApp(firebaseConfig);
    const db = firebase.database();

    // 2. CARGA DE DATOS EN TIEMPO REAL
    // Cada vez que tú o alguien más agregue una orden, aparecerá aquí automáticamente
    db.ref('ordenes').on('value', (snapshot) => {
        const tablaHistorial = document.getElementById("historial");
        tablaHistorial.innerHTML = "";
        const datos = snapshot.val();
        
        for (let id in datos) {
            let o = datos[id];
            let fila = tablaHistorial.insertRow();
            fila.innerHTML = `
                <td>${o.numOrden || 'N/A'}</td>
                <td>${o.equipo}</td>
                <td>${o.trabajo}</td>
                <td>${o.refacciones ? o.refacciones.join(", ") : ""}</td>
                <td>${o.responsable}</td>
                <td>${o.fecha}</td>
                <td><button onclick='crearPDF(${JSON.stringify(o)})'>PDF</button></td>
            `;
        }
    });

    function agregarFila(){
        let tabla = document.getElementById("refacciones");
        let fila = tabla.insertRow();
        fila.innerHTML = `<td><input class="id"></td><td><input class="nombre"></td><td><input class="cant" type="number"></td><td><input class="prov"></td>`;
    }

    // 3. GUARDAR EN LA NUBE
    function crearOrden(){
        let equipo = document.getElementById("equipo").value;
        let trabajo = document.getElementById("trabajo").value;
        let responsable = document.getElementById("responsable").value;
        let fecha = new Date().toLocaleString();

        let refacciones = [];
        document.querySelectorAll(".nombre").forEach((n,i)=>{
            if(n.value!="") refacciones.push(`${n.value} (${document.querySelectorAll(".cant")[i].value})`);
        });

        // Obtenemos el último número de orden para incrementarlo automáticamente
        db.ref('ordenes').limitToLast(1).once('value', (snapshot) => {
            let nuevoNum = 1;
            snapshot.forEach(child => { 
                nuevoNum = parseInt(child.val().numOrden) + 1; 
            });

            // Guardamos la nueva orden en la nube
            db.ref('ordenes').push({
                numOrden: nuevoNum,
                equipo: equipo,
                trabajo: trabajo,
                responsable: responsable,
                fecha: fecha,
                refacciones: refacciones
            });
        });

        // Limpiar campos después de enviar
        document.getElementById("equipo").value = "";
        document.getElementById("trabajo").value = "";
    }

    // FUNCIONES DE UTILIDAD (PDF Y EXCEL)
    function limpiarTexto(texto){ return texto.normalize("NFD").replace(/[\u0300-\u036f]/g, ""); }

    function crearPDF(o){
        const { jsPDF } = window.jspdf;
        let doc = new jsPDF();
        let logo = new Image();
        logo.src = "https://i.imgur.com/53mj1cB.png";
        logo.onload = function(){
            doc.addImage(logo,'PNG',15,10,35,18);
            doc.setFontSize(18);
            doc.text("ORDEN DE TRABAJO",105,20,null,null,"center");
            doc.autoTable({
                startY: 40,
                head: [["Campo", "Informacion"]],
                body: [
                    ["Numero de Orden", o.numOrden],
                    ["Equipo", limpiarTexto(o.equipo)],
                    ["Responsable", limpiarTexto(o.responsable)],
                    ["Fecha", o.fecha],
                    ["Trabajo Realizado", limpiarTexto(o.trabajo)]
                ],
                theme: "grid",
                headStyles: { fillColor: [128, 0, 32] }
            });
            doc.save("orden_"+o.numOrden+".pdf");
        }
    }

    function exportarExcel(){
        let wb = XLSX.utils.table_to_book(document.getElementById("tablaOrdenes"));
        XLSX.writeFile(wb,"ordenes_mantenimiento.xlsx");
    }
</script>
</body>
</html>