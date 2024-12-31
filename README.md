<!DOCTYPE html>
<html>
<head>
    <title>Mapa com Leaflet e OpenStreetMap</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
    <script src="https://unpkg.com/leaflet-omnivore@0.3.4/leaflet-omnivore.min.js"></script>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
        }
        #map {
            height: calc(100vh - 50px);
            margin-top: 50px;
        }
        .menu-bar {
            display: flex;
            background-color: #f1f1f1;
            padding: 10px;
            position: fixed;
            width: 100%;
            top: 0;
            z-index: 1000;
        }
        .menu-bar > div {
            margin-right: 20px;
            cursor: pointer;
            position: relative;
        }
        .submenu {
            display: none;
            position: absolute;
            background-color: #f9f9f9;
            box-shadow: 0px 8px 16px 0px rgba(0,0,0,0.2);
            z-index: 1001;
            top: 100%;
            left: 0;
            width: 200px;
        }
        .submenu div {
            padding: 12px 16px;
            cursor: pointer;
        }
        .menu-bar > div:hover .submenu {
            display: block;
        }
        .sidebar {
            display: block;
            position: fixed;
            top: 50px;
            right: 0;
            width: 300px;
            height: calc(100vh - 50px);
            background-color: #f1f1f1;
            box-shadow: -2px 0px 5px rgba(0,0,0,0.5);
            z-index: 1000;
            padding: 10px;
            overflow-y: auto;
        }
        .toolbar {
            display: none;
            position: fixed;
            bottom: 0;
            left: 0;
            width: 100%;
            background-color: #f1f1f1;
            padding: 10px;
            box-shadow: 0px -2px 5px rgba(0,0,0,0.5);
            z-index: 1000;
        }
        .sidebar-section {
            margin-bottom: 20px;
        }
        .sidebar-section h3 {
            margin-top: 0;
        }
        .search-bar {
            display: flex;
            margin-bottom: 10px;
        }
        .search-bar input {
            flex: 1;
            padding: 5px;
        }
        .search-bar button {
            padding: 5px;
        }
        .leaflet-control-zoom {
            position: fixed;
            bottom: 10px;
            left: 10px;
            z-index: 1000;
        }
        .folder {
            cursor: pointer;
        }
        .folder ul {
            display: none;
            list-style-type: none;
            padding-left: 20px;
        }
        .folder.open ul {
            display: block;
        }
    </style>
</head>
<body>
    <div class="menu-bar">
        <div>
            Arquivo
            <div class="submenu">
                <div onclick="document.getElementById('fileInput').click();">Abrir</div>
                <div onclick="saveFiles()">Salvar</div>
                <div onclick="window.print()">Imprimir</div>
            </div>
        </div>
        <div>
            Visualizar
            <div class="submenu">
                <div onclick="toggleSidebar()">Barra Lateral</div>
                <div onclick="toggleToolbar()">Barra de Ferramentas</div>
            </div>
        </div>
        <div>
            Adicionar
            <div class="submenu">
                <div onclick="createFolder()">Criar Pasta</div>
                <div onclick="addMarker()">Marcador</div>
                <div onclick="addPath()">Caminhos</div>
            </div>
        </div>
        <div>
            Ajuda
            <div class="submenu">
                <div onclick="showInstructions()">Instruções</div>
            </div>
        </div>
    </div>
    <div class="sidebar" id="sidebar">
        <div class="search-bar">
            <input type="text" id="searchInput" placeholder="Pesquisar...">
            <button onclick="searchMarkers(document.getElementById('searchInput').value)">Buscar</button>
        </div>
        <div class="sidebar-section">
            <h3>Lugares</h3>
            <ul id="fileList"></ul>
        </div>
        <div class="sidebar-section">
            <h3>Camadas</h3>
            <label><input type="checkbox" onclick="toggleMarkers(this.checked)" checked> Marcadores</label><br>
            <label><input type="checkbox" onclick="togglePaths(this.checked)" checked> Rotas</label><br>
            <label><input type="checkbox" onclick="toggleFiles(this.checked)" checked> Arquivos</label>
        </div>
    </div>
    <div class="toolbar" id="toolbar">
        <button onclick="addMarker()">Adicionar Marcador</button>
        <button onclick="addPath()">Adicionar Caminho</button>
    </div>
    <div id="map"></div>
    <input type="file" id="fileInput" style="display:none" accept=".kml,.kmz" onchange="handleFileUpload(event)" />

    <script>
        var map = L.map('map').setView([-23.55052, -46.633308], 13); // Coordenadas de São Paulo, Brasil

        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
        }).addTo(map);

        var layers = {};
        var markers = [];
        var paths = [];
        var files = [];

        function loadKmlLayer(kmlUrl, layerName) {
            var layer = omnivore.kml(kmlUrl)
                .on('ready', function() {
                    var bounds = this.getBounds();
                    if (bounds.isValid()) {
                        map.fitBounds(bounds);
                    } else {
                        console.error('Bounds are not valid.');
                    }
                    this.eachLayer(function(layer) {
                        if (layer.feature && layer.feature.properties) {
                            var popupContent = '<b>Informações:</b><br>';
                            for (var key in layer.feature.properties) {
                                if (layer.feature.properties.hasOwnProperty(key)) {
                                    popupContent += key + ': ' + layer.feature.properties[key] + '<br>';
                                }
                            }
                            layer.bindPopup(popupContent);
                            markers.push(layer);


                            // Alterar cor dos caminhos
                            if (layer.feature.geometry.type === 'LineString') {
                                if (layer.feature.properties.description.includes('operada: F')) {
                                    layer.setStyle({ color: 'orange' });
                                } else if (layer.feature.properties.description.includes('operada: A')) {
                                    layer.setStyle({ color: 'blue' });
                                }
                            }

                            if (layer.feature.geometry.type === 'LineString') {
                                if (layer.feature.properties.description.includes('Area: R')) {
                                    layer.setStyle({ color: 'green' });
                                } else if (layer.feature.properties.description.includes('Area: U')) {
                                    layer.setStyle({ color: 'gold' });
                                }
                            }

                            if (layer.feature.geometry.type === 'LineString') {
                                if (["Religador", "Chave Fusfvel", "CH. Fus. Repet.", "CH Fus Especial", "Seccionalizador", "CH Protecao Sub"].some(text => layer.feature.properties.description.includes(text))) {
                                    layer.setStyle({ color: 'purple' });
                                }
                            }

                           // Alterar ícones dos marcadores
                           if (layer.feature.geometry.type === 'Point') {
                                var iconUrlReligador = 'https://maps.google.com/mapfiles/kml/pal2/icon1.png';
                                var iconUrlBarra = 'https://maps.google.com/mapfiles/kml/pal3/icon53.png';
                                var iconUrlNome = 'https://maps.google.com/mapfiles/kml/pal2/icon10.png';
                                var iconUrlPadrao = 'https://maps.google.com/mapfiles/kml/shapes/placemark_circle.png';
                                var iconUrl;

                                // Verifica se as propriedades existem e se o texto do ícone contém as palavras específicas
                                if (layer.feature.properties && layer.feature.properties.description) {
                                    if (layer.feature.properties.description.includes('Religador')) {
                                        iconUrl = iconUrlReligador;
                                    } else if (layer.feature.properties.description.includes('Barra')) {
                                        iconUrl = iconUrlBarra;
                                    } else if (layer.feature.properties.description.includes('Nome')) {
                                        iconUrl = iconUrlNome;
                                    } else {
                                        iconUrl = iconUrlPadrao;
                                    }
                                } else {
                                    iconUrl = iconUrlPadrao;
                                }

                             
    var icon = L.icon({
    iconUrl: iconUrl,
    iconSize: [16, 16], // Escala 0.5
    iconAnchor: [8, 8],
    popupAnchor: [0, -8],
    className: 'custom-icon'
});

layer.setIcon(icon);
}
}
});
})
.on('error', function(e) {
console.error('Erro ao carregar KML:', e);
})
.addTo(map);
layers[layerName] = layer;
files.push(layer);
}

function toggleLayer(layerName, visible) {
if (layers[layerName]) {
if (visible) {
map.addLayer(layers[layerName]);
} else {
map.removeLayer(layers[layerName]);
}
}
}

function toggleMarkers(visible) {
markers.forEach(function(marker) {
if (visible) {
map.addLayer(marker);
} else {
map.removeLayer(marker);
}
});
}

function togglePaths(visible) {
paths.forEach(function(path) {
if (visible) {
map.addLayer(path);
} else {
map.removeLayer(path);
}
});
}

function toggleFiles(visible) {
files.forEach(function(file) {
if (visible) {
map.addLayer(file);
} else {
map.removeLayer(file);
}
});
}

function searchMarkers(query) {
markers.forEach(function(marker) {
if (marker.feature && marker.feature.properties) {
var match = false;
for (var key in marker.feature.properties) {
if (marker.feature.properties.hasOwnProperty(key) && marker.feature.properties[key].toString().toLowerCase().includes(query.toLowerCase())) {
match = true;
break;
}
}
if (match) {
marker.openPopup();
map.setView(marker.getLatLng(), 15);
} else {
marker.closePopup();
}
}
});
}

function handleFileUpload(event) {
var file = event.target.files[0];
var reader = new FileReader();
reader.onload = function(e) {
var kmlText = e.target.result;
var kmlUrl = URL.createObjectURL(new Blob([kmlText], { type: 'application/vnd.google-earth.kml+xml' }));
loadKmlLayer(kmlUrl, file.name);
addFileToList(file.name);
};
reader.readAsText(file);
}

function addFileToList(fileName) {
var fileList = document.getElementById('fileList');
var listItem = document.createElement('li');
var checkbox = document.createElement('input');
checkbox.type = 'checkbox';
checkbox.checked = true;
checkbox.onchange = function() {
toggleLayer(fileName, checkbox.checked);
};
listItem.appendChild(checkbox);
listItem.appendChild(document.createTextNode(fileName));
var deleteButton = document.createElement('button');
deleteButton.textContent = 'Excluir';
deleteButton.onclick = function() {
fileList.removeChild(listItem);
map.removeLayer(layers[fileName]);
delete layers[fileName];
};
listItem.appendChild(deleteButton);
fileList.appendChild(listItem);
}

function saveFiles() {
// Implementar a lógica para salvar os arquivos abertos
alert('Salvar arquivos ainda não implementado.');
}

function toggleSidebar() {
var sidebar = document.getElementById('sidebar');
if (sidebar.style.display === 'none' || sidebar.style.display === '') {
sidebar.style.display = 'block';
} else {
sidebar.style.display = 'none';
}
}

function toggleToolbar() {
var toolbar = document.getElementById('toolbar');
if (toolbar.style.display === 'none' || toolbar.style.display === '') {
toolbar.style.display = 'block';
} else {
toolbar.style.display = 'none';
}
}

function createFolder() {
var folderName = prompt('Digite o nome da nova pasta:');
if (folderName) {
var fileList = document.getElementById('fileList');
var folderItem = document.createElement('li');
folderItem.textContent = folderName;
var folderList = document.createElement('ul');
folderItem.appendChild(folderList);
fileList.appendChild(folderItem);
}
}

function addMarker() {
map.on('click', function(e) {
var marker = L.marker(e.latlng).addTo(map);
marker.bindPopup('Novo Marcador').openPopup();
markers.push(marker);
});
}

function addPath() {
var path = [];
map.on('click', function(e) {
path.push(e.latlng);
if (path.length > 1) {
var polyline = L.polyline(path, { color: 'red' }).addTo(map);
paths.push(polyline);
}
});
}

function showInstructions() {
alert('Instruções de uso:\n\n1. Use a aba "Arquivo" para abrir, salvar e imprimir arquivos KML/KMZ.\n2. Use a aba "Visualizar" para alternar a barra lateral e a barra de ferramentas.\n3. Use a aba "Adicionar" para criar pastas, adicionar marcadores e caminhos.\n4. Use a aba "Ajuda" para ver as instruções.');
}
</script>
</body>
</html>
