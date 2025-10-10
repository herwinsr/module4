Here’s the complete `index.jsx` (TransactionReport) with all changes:

```javascript
import React, { useState, useEffect } from 'react';
import dayjs from 'dayjs';
import { useDispatch } from 'react-redux';
import { addLoader, removeLoader } from '@redux/commonSlice';
import ReportContainer from '../common/ReportContainer';
import ApiService from '../Common/Services/ApiService';
import { TransactionReportValidation } from './TransactionReportValidation';

const TransactionReport = () => {
  const dispatch = useDispatch();
 
  const [formData, setFormData] = useState({
    partnerId: [],
    startTimeStamp: null,
    endTimeStamp: null,
    country: [],    
    filter: [],    
    txnType: []  
  });

  const [formErrors, setFormErrors] = useState({
    partnerId: '',
    startTimeStamp: '',
    endTimeStamp: '',
    country: '',
    filter: '',
    txnType: ''
  });

  const [partnerOptions, setPartnerOptions] = useState([]);
  const [countryOptions, setCountryOptions] = useState([]);

  const filterOptions = [
    { label: "ALL", value: "ALL" },
    { label: "All Transactions", value: "all_transactions" },
    { label: "Transaction Received Time", value: "received_time" },
    { label: "Transaction Processed Time", value: "processed_time" }
  ];

  const transactionTypeOptions = [
    { label: "ALL", value: "ALL" },
    { label: "Individual", value: "individual" },
    { label: "Business", value: "business" }
  ];

  useEffect(() => {
    fetchDropdownData();
  }, []);

  const fetchDropdownData = async () => {
    try {
      dispatch(addLoader("ACTION"));

      const partnerResponse = await ApiService.findAllPartnerGroups();
     
      const partners = (partnerResponse || [])
        .filter(item => item.partnerId)
        .map(item => ({
          label: item.partnerName,
          value: item.partnerId
        }));
     
      setPartnerOptions(partners);

      const countryResponse = await ApiService.findAllCountries();
     
      const countries = [
        { label: "All", value: "All" },
        ...(countryResponse || []).map(item => ({
          label: item.country,
          value: item.countryCode
        }))
      ];
     
      setCountryOptions(countries);
     
    } catch (error) {
      console.error('Error fetching dropdown data:', error);
      setPartnerOptions([]);
      setCountryOptions([{ label: "All", value: "All" }]);
    } finally {
      dispatch(removeLoader("ACTION"));
    }
  };

  const handleFieldChange = (fieldName, value) => {
    setFormData(prev => ({
      ...prev,
      [fieldName]: value
    }));
    
    setFormErrors(prev => ({
      ...prev,
      [fieldName]: TransactionReportValidation(fieldName, value, { ...formData, [fieldName]: value })
    }));
  };

  const isFormValid = () => {
    const temp = {};
    let isFormValid = true;
    
    Object.keys(formData).forEach((field) => {
      const status = TransactionReportValidation(field, formData[field], formData);
      temp[field] = status;
      if (status) isFormValid = false;
    });
    
    setFormErrors(temp);
    return isFormValid;
  };

  const handleDownload = async (actionName) => {
    if (!isFormValid()) {
      return;
    }

    try {
      dispatch(addLoader("ACTION"));
 
      const formattedData = {
        partnerId: formData.partnerId?.value || formData.partnerId,
        startTimeStamp: formData.startTimeStamp ? dayjs(formData.startTimeStamp).format('YYYY-MM-DD HH:mm:ss') : null,
        endTimeStamp: formData.endTimeStamp ? dayjs(formData.endTimeStamp).format('YYYY-MM-DD HH:mm:ss') : null,
        country: formData.country?.value || 'All',  
        filter: formData.filter?.value || 'ALL',  
        txnType: formData.txnType?.value || 'ALL'  
      };

      console.log('Formatted Data for API:', formattedData);

      switch (actionName) {
        case 'getPartnerFormat':
          await ApiService.downloadTransactionReport(formattedData);
          break;
        case 'getTopsFormat':
          await ApiService.downloadTransactionReportXLS(formattedData);
          break;
        case 'getCommissionFormat':
          await ApiService.downloadTransactionCommissionReport(formattedData);
          break;
        default:
          console.log('Unknown action:', actionName);
      }
    } catch (error) {
      console.error('Download failed:', error);
    } finally {
      dispatch(removeLoader("ACTION"));
    }
  };

  const handleReset = () => {
    setFormData({
      partnerId: null,
      startTimeStamp: null,
      endTimeStamp: null,
      country: null,
      filter: null,
      txnType: null
    });
    
    setFormErrors({
      partnerId: '',
      startTimeStamp: '',
      endTimeStamp: '',
      country: '',
      filter: '',
      txnType: ''
    });
  };

  const fields = [
    {
      name: 'partnerId',
      type: 'dropdown',
      label: 'Select Partner',
      required: true,
      multiple: true,
      options: partnerOptions
    },
    {
      name: 'startTimeStamp',
      type: 'datetime',
      label: 'Start Time',
      placeholder: 'Select Start Time',
      required: true
    },
    {
      name: 'endTimeStamp',
      type: 'datetime',
      label: 'End Time',
      placeholder: 'Select End Time',
      required: true,
      minDateTime: formData.startTimeStamp
    },
    {
      name: 'filter',
      type: 'dropdown',
      label: 'Filter Type',
      multiple: true,
      options: filterOptions
    },
    {
      name: 'country',
      type: 'dropdown',
      label: 'Receive Country',
      multiple: true,
      options: countryOptions
    },
    {
      name: 'txnType',
      type: 'dropdown',
      label: 'Transaction Type',
      multiple: true,
      options: transactionTypeOptions
    }
  ];

  const buttons = [
    {
      label: 'Get Partner Format',
      onClick: () => handleDownload('getPartnerFormat'),
      variant: 'contained'
    },
    {
      label: 'Get TOPS Format',
      onClick: () => handleDownload('getTopsFormat'),
      variant: 'contained'
    },
    {
      label: 'Get Commission Format',
      onClick: () => handleDownload('getCommissionFormat'),
      variant: 'contained'
    },
    {
      label: 'Reset',
      onClick: handleReset,
      variant: 'outlined'
    }
  ];

  return (
    <ReportContainer
      title="Transaction Report"
      headerTitle="Transaction Filters"
      fields={fields}
      formData={formData}
      formErrors={formErrors}
      handleFieldChange={handleFieldChange}
      buttons={buttons}
    />
  );
};

export default TransactionReport;
```

**Key changes made:**

1. ✅ Added `import { TransactionReportValidation } from './TransactionReportValidation';`
1. ✅ Added `formErrors` state
1. ✅ Updated `handleFieldChange` to validate on change
1. ✅ Added `isFormValid()` function (exactly like currency form)
1. ✅ Removed `validateTimeStamp` function (moved to validation file)
1. ✅ Updated `handleDownload` to use `isFormValid()`
1. ✅ Updated `handleReset` to clear errors
1. ✅ Added `formErrors={formErrors}` prop to `ReportContainer`