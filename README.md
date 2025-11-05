import React, { useState, useEffect, useRef } from 'react';
import {
  Ship,
  Plane,
  Package,
  MapPin,
  Calculator,
  FileText,
  Shield,
  Truck,
  Warehouse,
  DollarSign,
  Download,
  Mail
} from 'lucide-react';
import jsPDF from 'jspdf';
import html2canvas from 'html2canvas';

const App = () => {
  const [activeTab, setActiveTab] = useState('air');
  const [formData, setFormData] = useState({
    transportType: 'air',
    incoterm: 'FOB',
    originCountry: '',
    originCity: '',
    destinationCountry: 'Chile',
    destinationCity: '',
    cargoType: 'general',
    weight: '',
    volume: '',
    value: '',
    pickup: false,
    customsAgency: false,
    taxPayment: false,
    lastMile: false,
    insurance: false,
    warehousing: false,
    consolidation: false,
    wireTransfer: false,
    specialRequirements: '',
    expectedDate: ''
  });

  const [quoteResult, setQuoteResult] = useState(null);
  const quoteRef = useRef(); // Para capturar el contenido del PDF

  useEffect(() => {
    setFormData(prev => ({ ...prev, transportType: activeTab }));
  }, [activeTab]);

  // ... (mismos datos: originCities, destinationCities, incoterms, etc.)
  const originCities = {
    'China': ['Shanghai', 'Beijing', 'Guangzhou', 'Shenzhen', 'Tianjin', 'Qingdao', 'Ningbo', 'Xiamen'],
    'Estados Unidos': ['New York', 'Los Angeles', 'Chicago', 'Houston', 'Miami', 'Seattle', 'Atlanta', 'Dallas'],
    'Colombia': ['Bogotá', 'Medellín', 'Cali', 'Barranquilla', 'Cartagena'],
    'Perú': ['Lima', 'Arequipa', 'Trujillo', 'Chiclayo', 'Piura'],
    'Unión Europea': ['Amsterdam', 'Antwerp', 'Barcelona', 'Berlin', 'Brussels', 'Copenhagen', 'Dublin', 'Frankfurt', 'Hamburg', 'Helsinki', 'Lisbon', 'Madrid', 'Milan', 'Paris', 'Rotterdam', 'Stockholm']
  };

  const destinationCities = [
    'Arica', 'Iquique', 'Antofagasta', 'La Serena', 'Santiago', 'Valparaíso/Viña Del Mar', 'Rancagua', 'Curicó', 'Talca', 'Concepción', 'Chillán', 'Temuco', 'Puerto Montt', 'Coyhaique', 'Punta Arenas'
  ];

  const incoterms = [
    { value: 'EXW', label: 'EXW - En fábrica' },
    { value: 'FCA', label: 'FCA - Libre transportista' },
    { value: 'FOB', label: 'FOB - Libre a bordo' },
    { value: 'CFR', label: 'CFR - Costo y flete' },
    { value: 'CIF', label: 'CIF - Costo, seguro y flete' },
    { value: 'DAP', label: 'DAP - Entregado en lugar' },
    { value: 'DDP', label: 'DDP - Entregado derechos pagados' }
  ];

  const cargoTypes = [
    { value: 'general', label: 'Carga general' },
    { value: 'dangerous', label: 'Carga peligrosa' },
    { value: 'refrigerated', label: 'Refrigerada' },
    { value: 'oversized', label: 'Sobredimensionada' },
    { value: 'bulk', label: 'A granel' }
  ];

  const originCountries = ['China', 'Estados Unidos', 'Colombia', 'Perú', 'Unión Europea'];

  const handleInputChange = (field, value) => {
    setFormData(prev => {
      const newFormData = { ...prev, [field]: value };
      if (field === 'originCountry') {
        newFormData.originCity = '';
      }
      return newFormData;
    });
  };

  const calculateQuote = () => {
    const weight = parseFloat(formData.weight) || 0;
    const volume = parseFloat(formData.volume) || 0;
    const value = parseFloat(formData.value) || 0;

    let transportCost = 0;
    let chargeableWeight = null;
    let volumeUsed = null;
    let modeLabel = '';

    if (activeTab === 'air') {
      const volumetricWeight = volume * 167;
      chargeableWeight = Math.max(weight, volumetricWeight);
      transportCost = chargeableWeight * 4.5;
      modeLabel = 'Aéreo';
    } else {
      volumeUsed = volume;
      transportCost = volume * 80; // USD/m³
      modeLabel = 'Marítimo';
    }

    const servicesCost = {
      pickup: formData.pickup ? 150 : 0,
      customsAgency: formData.customsAgency ? 200 : 0,
      taxPayment: formData.taxPayment ? value * 0.15 : 0,
      lastMile: formData.lastMile ? 250 : 0,
      insurance: formData.insurance ? value * 0.02 : 0,
      warehousing: formData.warehousing ? 50 : 0,
    };

    const totalServices = Object.values(servicesCost).reduce((sum, cost) => sum + cost, 0);
    const totalCost = transportCost + totalServices;

    setQuoteResult({
      transportCost: Math.round(transportCost * 100) / 100,
      servicesCost: Math.round(totalServices * 100) / 100,
      totalCost: Math.round(totalCost * 100) / 100,
      breakdown: servicesCost,
      chargeableWeight,
      volumeUsed,
      modeLabel
    });
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    const { originCountry, originCity, destinationCity, weight, volume, value } = formData;

    if (!originCountry || !originCity || !destinationCity) {
      alert('Por favor complete todos los campos de ubicación');
      return;
    }
    if (!weight && !volume) {
      alert('Debe ingresar al menos el peso (kg) o el volumen (m³) de la carga');
      return;
    }
    if (activeTab === 'sea' && !volume) {
      alert('En transporte marítimo, el volumen (m³) es obligatorio');
      return;
    }
    if (!value) {
      alert('Por favor ingrese el valor de la mercadería (USD)');
      return;
    }

    calculateQuote();
  };

  // 📧 Enviar por email
  const handleEmail = () => {
    if (!quoteResult) return;

    const subject = encodeURIComponent(`Cotización de carga - CargoSud Pro`);
    const bodyLines = [
      `Cotización generada el ${new Date().toLocaleDateString('es-ES')}`,
      ``,
      `Origen: ${formData.originCity}, ${formData.originCountry}`,
      `Destino: ${formData.destinationCity}, Chile`,
      `Modo: ${quoteResult.modeLabel}`,
      `Incoterm: ${formData.incoterm}`,
      `Tipo de carga: ${cargoTypes.find(c => c.value === formData.cargoType)?.label || formData.cargoType}`,
      ``,
      `DETALLE DE COSTOS:`,
      `Transporte ${quoteResult.modeLabel}: $${quoteResult.transportCost.toLocaleString()}`,
      ...(quoteResult.breakdown.pickup > 0 ? [`Recogida en origen: $${quoteResult.breakdown.pickup.toLocaleString()}`] : []),
      ...(quoteResult.breakdown.customsAgency > 0 ? [`Agencia de aduanas: $${quoteResult.breakdown.customsAgency.toLocaleString()}`] : []),
      ...(quoteResult.breakdown.taxPayment > 0 ? [`Impuestos: $${quoteResult.breakdown.taxPayment.toLocaleString()}`] : []),
      ...(quoteResult.breakdown.lastMile > 0 ? [`Última milla: $${quoteResult.breakdown.lastMile.toLocaleString()}`] : []),
      ...(quoteResult.breakdown.insurance > 0 ? [`Seguro: $${quoteResult.breakdown.insurance.toLocaleString()}`] : []),
      ...(quoteResult.breakdown.warehousing > 0 ? [`Almacenamiento: $${quoteResult.breakdown.warehousing.toLocaleString()}`] : []),
      ``,
      `TOTAL: $${quoteResult.totalCost.toLocaleString()}`,
      ``,
      `Requisitos especiales: ${formData.specialRequirements || 'Ninguno'}`,
      `Fecha de embarque estimada: ${formData.expectedDate || 'No especificada'}`
    ];

    const body = encodeURIComponent(bodyLines.join('\n'));
    window.location.href = `mailto:?subject=${subject}&body=${body}`;
  };

  // 📄 Descargar PDF
  const handleDownloadPDF = () => {
    if (!quoteRef.current) return;

    const input = quoteRef.current;
    html2canvas(input, { scale: 2, useCORS: true }).then(canvas => {
      const imgData = canvas.toDataURL('image/png');
      const pdf = new jsPDF('p', 'mm', 'a4');
      const width = pdf.internal.pageSize.getWidth();
      const height = (canvas.height * width) / canvas.width;

      pdf.addImage(imgData, 'PNG', 0, 0, width, height);
      pdf.save(`cotizacion-cargosud-${new Date().toISOString().split('T')[0]}.pdf`);
    });
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100">
      {/* Header */}
      <header className="bg-white shadow-sm border-b">
        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-4">
          <div className="flex items-center justify-between">
            <div className="flex items-center space-x-3">
              <div className="bg-blue-600 p-2 rounded-lg">
                {activeTab === 'air' ? <Plane className="h-6 w-6 text-white" /> : <Ship className="h-6 w-6 text-white" />}
              </div>
              <div>
                <h1 className="text-2xl font-bold text-gray-900">CargoSud Pro</h1>
                <p className="text-sm text-gray-600">Cotización detallada de importaciones</p>
              </div>
            </div>
            <div className="flex space-x-1 bg-gray-100 p-1 rounded-lg">
              <button
                onClick={() => setActiveTab('air')}
                className={`flex items-center space-x-2 px-4 py-2 rounded-md transition-colors ${
                  activeTab === 'air' 
                    ? 'bg-blue-600 text-white' 
                    : 'text-gray-600 hover:text-gray-900'
                }`}
              >
                <Plane className="h-4 w-4" />
                <span>Aérea</span>
              </button>
              <button
                onClick={() => setActiveTab('sea')}
                className={`flex items-center space-x-2 px-4 py-2 rounded-md transition-colors ${
                  activeTab === 'sea' 
                    ? 'bg-blue-600 text-white' 
                    : 'text-gray-600 hover:text-gray-900'
                }`}
              >
                <Ship className="h-4 w-4" />
                <span>Marítima</span>
              </button>
            </div>
          </div>
        </div>
      </header>

      <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
        <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
          {/* Form */}
          <div className="lg:col-span-2">
            <div className="bg-white rounded-xl shadow-lg p-6">
              <div className="flex items-center space-x-3 mb-6">
                <Calculator className="h-6 w-6 text-blue-600" />
                <h2 className="text-xl font-semibold text-gray-900">Detalles de la Carga</h2>
              </div>
              <form onSubmit={handleSubmit} className="space-y-6">
                {/* ... (el formulario es idéntico al anterior, se omite por brevedad) */}
                {/* Origen/Destino */}
                <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-2">
                      <MapPin className="inline h-4 w-4 mr-1" />
                      Origen
                    </label>
                    <select
                      value={formData.originCountry}
                      onChange={(e) => handleInputChange('originCountry', e.target.value)}
                      className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                      required
                    >
                      <option value="">Seleccione país de origen</option>
                      {originCountries.map(country => (
                        <option key={country} value={country}>{country}</option>
                      ))}
                    </select>
                    <select
                      value={formData.originCity}
                      onChange={(e) => handleInputChange('originCity', e.target.value)}
                      className="w-full mt-2 px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                      disabled={!formData.originCountry}
                      required
                    >
                      <option value="">Seleccione ciudad de origen</option>
                      {formData.originCountry && originCities[formData.originCountry]?.map(city => (
                        <option key={city} value={city}>{city}</option>
                      ))}
                    </select>
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-2">
                      <MapPin className="inline h-4 w-4 mr-1" />
                      Destino
                    </label>
                    <select
                      value={formData.destinationCountry}
                      disabled
                      className="w-full px-3 py-2 border border-gray-300 rounded-md bg-gray-100 cursor-not-allowed"
                    >
                      <option>Chile</option>
                    </select>
                    <select
                      value={formData.destinationCity}
                      onChange={(e) => handleInputChange('destinationCity', e.target.value)}
                      className="w-full mt-2 px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                      required
                    >
                      <option value="">Seleccione ciudad de destino</option>
                      {destinationCities.map(city => (
                        <option key={city} value={city}>{city}</option>
                      ))}
                    </select>
                  </div>
                </div>

                {/* Incoterm y Tipo de Carga */}
                <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-2">
                      Incoterm®
                    </label>
                    <select
                      value={formData.incoterm}
                      onChange={(e) => handleInputChange('incoterm', e.target.value)}
                      className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                    >
                      {incoterms.map(term => (
                        <option key={term.value} value={term.value}>{term.label}</option>
                      ))}
                    </select>
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-2">
                      Tipo de Carga
                    </label>
                    <select
                      value={formData.cargoType}
                      onChange={(e) => handleInputChange('cargoType', e.target.value)}
                      className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                    >
                      {cargoTypes.map(type => (
                        <option key={type.value} value={type.value}>{type.label}</option>
                      ))}
                    </select>
                  </div>
                </div>

                {/* Detalles de carga */}
                <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">
                      Peso (kg)
                    </label>
                    <input
                      type="number"
                      min="0"
                      step="0.1"
                      placeholder="Peso"
                      value={formData.weight}
                      onChange={(e) => handleInputChange('weight', e.target.value)}
                      className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                    />
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">
                      Volumen (m³)
                    </label>
                    <input
                      type="number"
                      min="0"
                      step="0.01"
                      placeholder="Volumen"
                      value={formData.volume}
                      onChange={(e) => handleInputChange('volume', e.target.value)}
                      className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                      required={activeTab === 'sea'}
                    />
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">
                      Valor (USD)
                    </label>
                    <input
                      type="number"
                      min="0"
                      step="0.01"
                      placeholder="Valor"
                      value={formData.value}
                      onChange={(e) => handleInputChange('value', e.target.value)}
                      className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                      required
                    />
                  </div>
                </div>

                {/* Servicios */}
                <div>
                  <h3 className="text-lg font-medium text-gray-900 mb-4 flex items-center">
                    <FileText className="h-5 w-5 mr-2 text-blue-600" />
                    Servicios Adicionales
                  </h3>
                  <div className="grid grid-cols-1 md:grid-cols-2 gap-3">
                    {[
                      { key: 'pickup', label: 'Recogida en origen', icon: Truck },
                      { key: 'customsAgency', label: 'Agencia de aduanas', icon: FileText },
                      { key: 'taxPayment', label: 'Pago de impuestos', icon: DollarSign },
                      { key: 'lastMile', label: 'Última milla', icon: Truck },
                      { key: 'insurance', label: 'Seguro de carga', icon: Shield },
                      { key: 'warehousing', label: 'Almacenamiento', icon: Warehouse },
                      { key: 'consolidation', label: 'Consolidación', icon: Warehouse },
                      { key: 'wireTransfer', label: 'Pago a proveedor', icon: DollarSign }
                    ].map(service => (
                      <label key={service.key} className="flex items-center space-x-3 cursor-pointer">
                        <input
                          type="checkbox"
                          checked={formData[service.key]}
                          onChange={(e) => handleInputChange(service.key, e.target.checked)}
                          className="h-4 w-4 text-blue-600 focus:ring-blue-500 border-gray-300 rounded"
                        />
                        <service.icon className="h-4 w-4 text-gray-600" />
                        <span className="text-sm text-gray-700">{service.label}</span>
                      </label>
                    ))}
                  </div>
                </div>

                {/* Detalles adicionales */}
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-2">
                    Requisitos Especiales
                  </label>
                  <textarea
                    rows={3}
                    placeholder="Manejo especial, documentación adicional, etc."
                    value={formData.specialRequirements}
                    onChange={(e) => handleInputChange('specialRequirements', e.target.value)}
                    className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                  />
                </div>

                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-2">
                    Fecha de Embarque Esperada
                  </label>
                  <input
                    type="date"
                    value={formData.expectedDate}
                    onChange={(e) => handleInputChange('expectedDate', e.target.value)}
                    className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                  />
                </div>

                <button
                  type="submit"
                  className="w-full bg-blue-600 text-white py-3 px-4 rounded-lg font-medium hover:bg-blue-700 transition-colors focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2"
                >
                  Calcular Cotización
                </button>
              </form>
            </div>
          </div>

          {/* Resultado */}
          <div className="space-y-6">
            {quoteResult ? (
              <div ref={quoteRef} className="bg-white rounded-xl shadow-lg p-6 sticky top-8">
                <div className="flex items-center space-x-3 mb-4">
                  <DollarSign className="h-6 w-6 text-green-600" />
                  <h2 className="text-xl font-semibold text-gray-900">Cotización Detallada</h2>
                </div>

                {/* Botones de acción */}
                <div className="flex space-x-2 mb-4">
                  <button
                    onClick={handleDownloadPDF}
                    className="flex-1 flex items-center justify-center space-x-1 bg-gray-800 text-white py-2 px-3 rounded-md text-sm hover:bg-gray-700 transition"
                  >
                    <Download className="h-4 w-4" />
                    <span>PDF</span>
                  </button>
                  <button
                    onClick={handleEmail}
                    className="flex-1 flex items-center justify-center space-x-1 bg-red-600 text-white py-2 px-3 rounded-md text-sm hover:bg-red-700 transition"
                  >
                    <Mail className="h-4 w-4" />
                    <span>Email</span>
                  </button>
                </div>

                {/* Contenido de la cotización (igual que antes) */}
                <div className="space-y-4">
                  <div className="bg-blue-50 rounded-lg p-4">
                    {quoteResult.chargeableWeight !== null ? (
                      <>
                        <div className="flex justify-between items-center mb-2">
                          <span className="text-sm font-medium text-blue-800">Peso facturable (Aéreo)</span>
                          <span className="text-lg font-semibold text-blue-900">
                            {quoteResult.chargeableWeight.toFixed(2)} kg
                          </span>
                        </div>
                      </>
                    ) : (
                      <>
                        <div className="flex justify-between items-center mb-2">
                          <span className="text-sm font-medium text-blue-800">Volumen (Marítimo)</span>
                          <span className="text-lg font-semibold text-blue-900">
                            {quoteResult.volumeUsed.toFixed(2)} m³
                          </span>
                        </div>
                      </>
                    )}
                    <div className="text-xs text-blue-700 mt-1">
                      {formData.originCity}, {formData.originCountry} → {formData.destinationCity}, Chile
                    </div>
                  </div>

                  <div className="space-y-3">
                    <div className="flex justify-between">
                      <span className="text-gray-600">Transporte {quoteResult.modeLabel}</span>
                      <span className="font-medium">${quoteResult.transportCost.toLocaleString()}</span>
                    </div>
                    {quoteResult.breakdown.pickup > 0 && (
                      <div className="flex justify-between">
                        <span className="text-gray-600">Recogida en origen</span>
                        <span className="font-medium">${quoteResult.breakdown.pickup.toLocaleString()}</span>
                      </div>
                    )}
                    {quoteResult.breakdown.customsAgency > 0 && (
                      <div className="flex justify-between">
                        <span className="text-gray-600">Agencia de aduanas</span>
                        <span className="font-medium">${quoteResult.breakdown.customsAgency.toLocaleString()}</span>
                      </div>
                    )}
                    {quoteResult.breakdown.taxPayment > 0 && (
                      <div className="flex justify-between">
                        <span className="text-gray-600">Impuestos</span>
                        <span className="font-medium">${quoteResult.breakdown.taxPayment.toLocaleString()}</span>
                      </div>
                    )}
                    {quoteResult.breakdown.lastMile > 0 && (
                      <div className="flex justify-between">
                        <span className="text-gray-600">Última milla</span>
                        <span className="font-medium">${quoteResult.breakdown.lastMile.toLocaleString()}</span>
                      </div>
                    )}
                    {quoteResult.breakdown.insurance > 0 && (
                      <div className="flex justify-between">
                        <span className="text-gray-600">Seguro</span>
                        <span className="font-medium">${quoteResult.breakdown.insurance.toLocaleString()}</span>
                      </div>
                    )}
                    {quoteResult.breakdown.warehousing > 0 && (
                      <div className="flex justify-between">
                        <span className="text-gray-600">Almacenamiento</span>
                        <span className="font-medium">${quoteResult.breakdown.warehousing.toLocaleString()}</span>
                      </div>
                    )}
                  </div>

                  <div className="border-t pt-4">
                    <div className="flex justify-between items-center">
                      <span className="text-lg font-semibold text-gray-900">Total</span>
                      <span className="text-2xl font-bold text-green-600">
                        ${quoteResult.totalCost.toLocaleString()}
                      </span>
                    </div>
                  </div>

                  <button className="w-full bg-green-600 text-white py-3 px-4 rounded-lg font-medium hover:bg-green-700 transition-colors">
                    Solicitar Cotización Oficial
                  </button>
                </div>
              </div>
            ) : (
              <div className="bg-white rounded-xl shadow-lg p-6">
                <div className="text-center py-8">
                  <Package className="h-12 w-12 text-gray-400 mx-auto mb-4" />
                  <h3 className="text-lg font-medium text-gray-900 mb-2">Cotización Pendiente</h3>
                  <p className="text-gray-600">Complete el formulario y haga clic en "Calcular Cotización" para ver los costos detallados.</p>
                </div>
              </div>
            )}

            {/* Info Card */}
            <div className="bg-gradient-to-r from-blue-500 to-indigo-600 rounded-xl shadow-lg p-6 text-white">
              <h3 className="text-lg font-semibold mb-2">¿Por qué elegirnos?</h3>
              <ul className="space-y-2 text-sm">
                <li className="flex items-start">
                  <div className="w-1.5 h-1.5 bg-white rounded-full mt-2 mr-3"></div>
                  <span>Cotizaciones transparentes y detalladas</span>
                </li>
                <li className="flex items-start">
                  <div className="w-1.5 h-1.5 bg-white rounded-full mt-2 mr-3"></div>
                  <span>Asesoramiento experto en Incoterms®</span>
                </li>
                <li className="flex items-start">
                  <div className="w-1.5 h-1.5 bg-white rounded-full mt-2 mr-3"></div>
                  <span>Red global de socios confiables</span>
                </li>
                <li className="flex items-start">
                  <div className="w-1.5 h-1.5 bg-white rounded-full mt-2 mr-3"></div>
                  <span>Seguimiento en tiempo real de su carga</span>
                </li>
              </ul>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
};

export default App;
