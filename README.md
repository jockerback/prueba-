<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Inventario Inteligente - iPhone</title>
    <!-- Tailwind CSS para un diseño limpio y moderno -->
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-slate-50 text-slate-800 min-p-4">

    <div class="max-w-md mx-auto bg-white rounded-3xl shadow-xl p-6 mt-6 border border-slate-100">
        <div class="text-center mb-6">
            <span class="bg-blue-100 text-blue-700 text-xs font-bold px-3 py-1 rounded-full uppercase tracking-wider">IA Vision</span>
            <h1 class="text-2xl font-black text-slate-900 mt-2">Inventario Móvil</h1>
            <p class="text-xs text-slate-500 mt-1">Toma una foto de tus productos para registrarlos automáticamente.</p>
        </div>
        
        <!-- Botón de la cámara nativa para iOS -->
        <label class="block w-full bg-blue-600 hover:bg-blue-700 active:scale-95 text-white text-center py-4 rounded-2xl font-bold cursor-pointer mb-6 shadow-lg shadow-blue-600/30 transition-all flex items-center justify-center gap-2">
            <span class="text-xl">📸</span> Tomar Foto del Producto
            <input type="file" accept="image/*" capture="environment" id="cameraInput" class="hidden" onchange="handleImage(event)">
        </label>

        <!-- Contenedor de la Tabla -->
        <div class="overflow-x-auto rounded-2xl border border-slate-200">
            <table class="w-full text-left border-collapse">
                <thead class="bg-slate-100 text-xs uppercase text-slate-600 tracking-wider">
                    <tr>
                        <th class="p-3">Nombre</th>
                        <th class="p-3">Marca</th>
                        <th class="p-3">Modelo</th>
                        <th class="p-3 text-center">Cant.</th>
                    </tr>
                </thead>
                <tbody id="inventoryTable" class="text-sm divide-y divide-slate-100">
                    <tr>
                        <td colspan="4" class="text-center p-6 text-slate-400 italic">No hay registros aún. ¡Toma una foto!</td>
                    </tr>
                </tbody>
            </table>
        </div>
        
        <!-- Botón para limpiar registros -->
        <div class="mt-4 text-center">
            <button onclick="clearTable()" class="text-xs text-red-500 hover:underline font-medium">Limpiar tabla</button>
        </div>
    </div>

    <script>
        // Configura tu clave de API de Gemini aquí
        const GEMINI_API_KEY = "TU_API_KEY_DE_GEMINI";

        async function handleImage(event) {
            const file = event.target.files[0];
            if (!file) return;

            const tbody = document.getElementById('inventoryTable');
            tbody.innerHTML = `<tr><td colspan="4" class="text-center p-6 text-blue-600 font-medium animate-pulse">Analizando imagen con Gemini...</td></tr>`;

            try {
                // Convertir la imagen capturada a Base64
                const base64Data = await convertFileToBase64(file);

                // Llamada a la API de Gemini (Modelo Flash)
                const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${GEMINI_API_KEY}`, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify({
                        contents: [{
                            parts: [
                                {
                                    text: "Analiza esta imagen y detecta los productos u objetos visibles. Devuelve la información exclusivamente en un formato de arreglo JSON válido, donde cada objeto tenga exactamente las siguientes claves: 'nombre' (texto descriptivo del producto), 'marca' (texto o 'Desconocida'), 'modelo' (texto o 'Genérico') y 'cantidad' (número de unidades visibles). No agregues texto adicional ni formato markdown extra, solo el JSON puro."
                                },
                                {
                                    inline_data: {
                                        mime_type: file.type,
                                        data: base64Data
                                    }
                                }
                            ]
                        }]
                    })
                });

                const data = await response.json();
                
                if (!data.candidates || data.candidates.length === 0) {
                    throw new Error("No se obtuvo respuesta de la IA.");
                }

                const aiText = data.candidates[0].content.parts[0].text;
                
                // Limpiar posibles marcas de código markdown en la respuesta
                const cleanJsonText = aiText.replace(/```json/g, '').replace(/```/g, '').trim();
                const detectedItems = JSON.parse(cleanJsonText);

                // Si es el primer elemento válido, limpiamos el mensaje de "No hay registros"
                if (tbody.querySelector('.italic')) {
                    tbody.innerHTML = '';
                }

                // Renderizar cada producto detectado en la tabla
                detectedItems.forEach(item => {
                    tbody.innerHTML += `
                        <tr class="hover:bg-slate-50 transition">
                            <td class="p-3 font-semibold text-slate-700">${item.nombre}</td>
                            <td class="p-3 text-slate-600">${item.marca}</td>
                            <td class="p-3 text-slate-500">${item.modelo}</td>
                            <td class="p-3 text-center font-bold text-blue-600">${item.cantidad}</td>
                        </tr>
                    `;
                });

            } catch (error) {
                console.error(error);
                tbody.innerHTML = `<tr><td colspan="4" class="text-center p-6 text-red-500 font-medium">Error al procesar la imagen. Inténtalo de nuevo.</td></tr>`;
            }
            
            // Limpiar el input para permitir tomar otra foto consecutiva
            event.target.value = '';
        }

        // Función auxiliar para convertir archivo a Base64
        function convertFileToBase64(file) {
            return new Promise((resolve, reject) => {
                const reader = new FileReader();
                reader.readAsDataURL(file);
                reader.onload = () => {
                    const base64String = reader.result.split(',')[1];
                    resolve(base64String);
                };
                reader.onerror = error => reject(error);
            });
        }

        // Función para vaciar la tabla
        function clearTable() {
            const tbody = document.getElementById('inventoryTable');
            tbody.innerHTML = `<tr><td colspan="4" class="text-center p-6 text-slate-400 italic">No hay registros aún. ¡Toma una foto!</td></tr>`;
        }
    </script>
</body>
</html>
