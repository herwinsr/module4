/* eslint-disable no-unused-vars */
/* eslint-disable react-hooks/exhaustive-deps */
import { useEffect, useState, useContext, useCallback, useMemo, useRef } from "react";
import { useNavigate, useSearchParams } from "react-router-dom";
import DataGridCommponent from "@/common/DataGridComponent/DataGridCommponent";
import {
  Box,
  IconButton,
  InputAdornment,
  Stack,
  Tooltip,
  Menu,
  MenuItem,
  CircularProgress,
  Popover,
  Backdrop,
  TextareaAutosize,
  Grid2,
} from "@mui/material";
import CommonModal from "@/common/Modal/CommonModal";
import CustomRadioButton from "@/common/RadioButton/CustomRadioButton";
import CommonButton from "@/common/Button/CommonButton";
import AddNewContent from "../Modal-Content/AddNewRate";
import ExtendValidity from "../Modal-Content/ExtendValidity";
import ExtendAndAddNew from "../Modal-Content/ExtendCurrentRateAndAddNewRate";
import ExtendCurrentRate from "../Modal-Content/ExpireCurrentRate";
import CustomPaginationComponent from "@/common/Pagination/CustomPaginationComponent";
import { Colors } from "@/theme/colors";
import {
  AddRateIcon,
  DeleteIcon,
  DownloadIcon,
  MoreIcon,
  SearchIcon,
  UpdateArrowIcon,
} from "../../../../assets/Icons";
import TextFieldComponent from "@/common/TextField/TextFieldComponent";
import { DataContext } from "../../../../context/DataContext";
import ResultPerPageComponent from "@/common/Pagination/ResultPerPageComponent";
import MultiSelectDropdown from "@/common/MultiselectDropdown.js/MultiSelect";
import { toast } from "react-toastify";
import {
  DateTimeFormat,
  displayFormattedRate,
  downloadData,
  removeCommas,
  displayToast,
  updateMetadata,
  sortAndMapArray,
  useMasterValuesByType,
  filterMVByType,
  useDefaultValueByKey,
  getStatusColor,
  sortAndMapArrayForAutocomplete,
  getCurrentGmtTime,
  getMappedValue,
  createMasterValueMap,
} from "@/utils/helperFunctions";
import {
  active,
  ALL_TEXT,
  COMMON_MASTER_LABELS,
  DATE_FORMAT,
  DealsMsgs,
  DEBOUNCE_DELAY,
  DEFAULT_FEED_TYPE,
  DEFAULT_OPTION_COUNTRY,
  DEFAULT_OPTION_PARTNER,
  DEFAULT_STATUS,
  DEFAULT_TEXT,
  ERROR_MESSAGES,
  // EXCLUDED_FEED_TYPE,
  ForexData,
  RadioButtonLabel as RadioBtnLabels,
  sourceRateMessage,
  STANDARD_TRANSACTION_TYPE,
  TimeBaseBulkTable,
  TimeBased,
} from "@/utils/ConstantLabel/Constant";
import BasicModal from "@/common/Modal/Modal";
import useDebouncer from "@/hooks/useDebouncer";
import { numericSearchFieldValid } from "../../../../utils/CommonValidation";
import TimeBasedOfferRateAPIs from "@/services-(general)/Treasury-Api/TimeBaseAddnewRateApiCall";
import DealBasedAPIs from "@/services-(general)/Treasury-Api/DealBaseApi";
import DateRangeComp from "@/common/DateAndTime/DateRangeComp";
import { format } from "date-fns";
import { useDispatch, useSelector } from "react-redux";
import { setForexType } from "@/redux/commonSlice";
import useTreasury from "../../../../hooks/useTreasury";
import { menuIDMap } from "../../../../utils/constants";
import { Download as WhiteDownloadIcon } from "@mui/icons-material";
import { setFeedType, setStatusValue } from "../../../../redux/commonSlice";
import ReusableTable from "./ReusableTable";
import axios from "axios";
import { paths } from "../../../../routes/paths";
import { Decrypt, Encrypt } from "@utils/Crypto";
import ForexApiCall from "@/services-(general)/Treasury-Api/ForexApiCall";
import moment from "moment";
import DateTimeRangePickerDialog from "@/common/DateAndTime/DateTimePickerDialog";
import dayjs from "dayjs";
import utc from "dayjs/plugin/utc";
import EventIcon from "@mui/icons-material/Event";
import CustomBackdrop from "@components/common/modal/Backdrop";

const TimeBaseOfferRateTable = ({ source }) => {
  // Import necessary modules and hooks
  const navigate = useNavigate();
  const [searchParams, setSearchParams] = useSearchParams();

  // Import APIs
  const {
    getTimeBase,
    getOfferRateById,
    getPartnerDropDown,
    getAllUserAccounts,
    updateTimeBaseRate,
  } = DealBasedAPIs();
  const { countryModalList, timeBaseDownload } = TimeBasedOfferRateAPIs();
  const { getFxFeedName, timeBaseDefaultPartner } = ForexApiCall();
  // State variables
  const [openModal, setOpenModal] = useState(false);
  const {
    configApiData,
    resultPerPage,
    sourceCurrency,
    setSourceCurrency,
    destinationCurrency,
    setDestinationCurrency,
    dateFilter,
    setDateFilter,
    destCountryCode,
    setdestCountryCode,
    sendPartnerId,
    setSendPartnerId,
    timeBaseSearch,
    setTimeBaseSearch,
    instrType,
    setInstrType,
    timeBaseIntrType,
    setTimeBaseInstrType,
    txnType,
    setTxnType,
    timeBaseStatus,
    setTimeBaseStatus,
    dropdownState,
    setDropdownState,
  } = useContext(DataContext);
  const { permissions } = useTreasury(menuIDMap?.TREASURY_OFFER_RATE);
  const RadioButtonLabel = [...RadioBtnLabels, permissions.hasDeleteAccess ? "Delete Rate" : null].filter(Boolean);
  const [selectedValue, setSelectedValue] = useState(RadioButtonLabel[0]);
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [isMenuOpen, setIsMenuOpen] = useState(false);
  const [fxFeed, setFxFeed] = useState(null);
  const [timebasedata, setTimeBaseData] = useState([]);
  const [loading, setLoading] = useState(false);
  const [deleteAll, setDeleteAll] = useState({
    loading: false,
    modal: false,
  });
  const [page, setPage] = useState(1);
  const [limit, setLimit] = useState(configApiData?.pageLimit);
  const [totalRowsCount, setTotalRowsCount] = useState();
  const [selectDate, setSelectDate] = useState("");
  const [selectedRowData, setSelectedRowData] = useState(null);
  const [rowDataToUpdate, setRowDataToUpdate] = useState(null);
  const [countryListVal, setCountryListVal] = useState([]);
  const [viewCountry, setViewCountry] = useState(false);
  const [viewPartners, setViewPartners] = useState(false);
  const [partnersList, setPartnersList] = useState([]);
  const [isOpen, setIsOpen] = useState(false);
  const [sortInitialState, setSortInitialState] = useState([]);
  const searchQueryWithoutCommas = removeCommas(timeBaseSearch);
  const debouncerValue = useDebouncer(searchQueryWithoutCommas, DEBOUNCE_DELAY);
  const [orderBy, setOrderBy] = useState("");
  const [sortBy, setSortBy] = useState("");
  const [isLoading, setIsLoading] = useState(false);
  const [isModalLoading, setIsModalLoading] = useState(false);
  const [isDataAvailable, setIsDataAvailable] = useState(false);
  const [anchorEl, setAnchorEl] = useState(null);
  const [maxDateLimit, setMaxDateLimit] = useState(null);
  const [userList, setUserList] = useState([]);
  const [userListChange, setUserListChange] = useState(null);
  const [feedTypeChange, setFeedTypeChange] = useState([]);
  const [instrTypeChange, setInstrTypeChange] = useState(null);
  const [txnTypeChange, setTxnTypeChange] = useState(null);
  const [selectedRange, setSelectedRange] = useState({
    startDate: dateFilter?.fromDate ? new Date(dateFilter?.fromDate) : null,
    endDate: dateFilter?.toDate ? new Date(dateFilter?.toDate) : null,
  });
  const [selectedRows, setSelectedRows] = useState([]);
  const [isDeletePopupOpen, setIsDeletePopupOpen] = useState(false);
  const [isProcessing, setIsProcessing] = useState(false);
  const [isRateByIdFailed, setIsRateByIdFailed] = useState(false);
  // Redux hooks and selectors
  const dispatch = useDispatch();
  const forexTypeValues = useMasterValuesByType("FOREX_TYPE");
  const forexType = useSelector((state) => state.commonReducer.forexType);
  const feedTypes = useSelector((state) => filterMVByType(state, "FEED_TYPE"));
  const feedTypeRaw = useMasterValuesByType("FEED_TYPE");
  const feedType = useSelector((state) => state.commonReducer.feedType);
  const feedTypeOpt = feedTypeRaw
    ? [...feedTypeRaw]
      .sort((a, b) => a.master_value_display_order - b.master_value_display_order)
      .map((type) => ({
        value: type.master_value_value,
        label: type.master_value_name,
      }))
    : [];

  const statusTypesRaw = useMasterValuesByType("OFFER_RATE_STATUS");
  const statusOpt = statusTypesRaw
    ? [...statusTypesRaw]
      .sort((a, b) => a.master_value_display_order - b.master_value_display_order)
      .map((type) => ({
        value: type.master_value_value,
        label: type.master_value_name,
      }))
    : [];

  const statusTypes = useSelector((state) => filterMVByType(state, "OFFER_RATE_STATUS"));
  const statusType = useSelector((state) => state.commonReducer.statusValue);

  dayjs.extend(utc);
  const [isDialogOpen, setIsDialogOpen] = useState(false);
  const [startDateTime, setStartDateTime] = useState(dayjs.utc().set("second", 0));
  const [endDateTime, setEndDateTime] = useState(dayjs.utc().add(1, "day").set("second", 59));
  const [isDateTimeSelected, setIsDateTimeSelected] = useState(false);
  const now = moment().utc();
  const [startDate, setStartDate] = useState(null);
  const [endDate, setEndDate] = useState(null);
  const [partnerList, setPartnerList] = useState([]);

  //  currency & country list we are fetcthing from redux
  const { currencyList, countryList } = useSelector((state) => state.commonReducer);
  const RATE_EXPIRATION_TIME = useDefaultValueByKey("RATE_EXPIRATION_TIME");
  const dateLimit = useDefaultValueByKey("TREASURY_MAX_DATE_LIMIT");
  const expHour = useDefaultValueByKey("DELETE_EXPIRED_SOURCE_RATE_HOUR");

  // Use effect to set max date limit
  useEffect(() => {
    const maxLimitValue = parseInt(dateLimit);
    if (!isNaN(maxLimitValue)) {
      setMaxDateLimit(maxLimitValue);
    }
  }, [dateLimit]);

  // Use effect to handle forex type
  useEffect(() => {
    const modifiedForexTypes = [...forexTypeValues];

    updateMetadata(modifiedForexTypes, forexType, setForexType, dispatch);
  }, [forexTypeValues, dispatch, forexType]);

  // Use effect to handle feed types
  useEffect(() => {
    updateMetadata(feedTypes, feedType, setFeedType, dispatch);
  }, [feedTypes, dispatch, feedType]);

  // Use effect to handle status types
  useEffect(() => {
    const modifiedStatusTypes = [...statusTypes];

    updateMetadata(modifiedStatusTypes, statusType, setStatusValue, dispatch);
  }, [statusTypes, dispatch, statusType]);

  // State variables for handling dropdown options list
  const currency = useMemo(() => {
    return currencyList
      ? sortAndMapArray(currencyList, "currency_abbreviation", {
        valueProperty: "currency_code",
        labelProperty1: "currency_abbreviation",
        labelProperty2: "currency_code",
      })
      : [];
  }, [currencyList]);
  const Partner = useMemo(() => {
    return partnerList
      ? sortAndMapArray(partnerList, "partner_name", {
        valueProperty: "partner_id",
        labelProperty1: "partner_name",
        labelProperty2: "partner_short_name",
      })
      : [];
  }, [partnerList]);
  const country = useMemo(() => {
    return countryList
      ? sortAndMapArray(countryList, "country_name", {
        valueProperty: "country_code",
        labelProperty1: "country_name",
        labelProperty2: "country_code",
      })
      : [];
  }, [countryList]);

  let instrumentTypesRaw = useMasterValuesByType("FX_INSTRUMENT_TYPE");
  instrumentTypesRaw = instrumentTypesRaw?.map((v) => ({
    ...v,
    master_value_value: v.master_value_value,
  }));
  // fetching the instrument type from the master value
  const instrumentTypes = useMemo(
    () =>
      instrumentTypesRaw
        ? sortAndMapArray(instrumentTypesRaw, "master_value_display_order", {
          valueProperty: "master_value_value",
          labelProperty1: "master_value_name",
        })
        : [],
    [instrumentTypesRaw]
  );

  let transactionTypesRaw = useMasterValuesByType("STANDARD_TRANSACTION_TYPE");
  // transactionTypesRaw = transactionTypesRaw?.map((v) => ({
  //   ...v,
  //   master_value_value: v.master_value_name?.toLowerCase(),
  // }));
  const transactionTypes = useMemo(() => {
    if (!transactionTypesRaw) return [];
    return transactionTypesRaw
      .sort((a, b) => (a.master_value_display_order ?? 0) - (b.master_value_display_order ?? 0))
      .map((v) => ({
        value: v.master_value_value,
        label: v.master_value_name,
      }));
  }, [transactionTypesRaw]);
  console.log(transactionTypes, "transactionTypes");
  let schemeRaw = useMasterValuesByType("SCHEMES");
  transactionTypesRaw = transactionTypesRaw?.map((v) => ({
    ...v,
    master_value_value: v.master_value_name?.toLowerCase(),
  }));
  const schemesList = useMemo(() => {
    if (schemeRaw?.length) {
      return sortAndMapArrayForAutocomplete(schemeRaw, "master_value_display_order", {
        valueProperty: "master_value_value",
        labelProperty1: "master_value_name",
      });
    }
    return [];
  }, [schemeRaw]);
  const instrumentTypeMap = useMemo(
    () => createMasterValueMap(instrumentTypesRaw),
    [instrumentTypesRaw]
  );
  const cardKey = getMappedValue(instrumentTypeMap, COMMON_MASTER_LABELS.INSTRUMENT.CARD);
  const bankKey = getMappedValue(instrumentTypeMap, COMMON_MASTER_LABELS.INSTRUMENT.BANK);
  const walletKey = getMappedValue(instrumentTypeMap, COMMON_MASTER_LABELS.INSTRUMENT.WALLET);
  // on row selection
  const handleRowSelection = (rows) => {
    setSelectedRows(rows);
  };

  // Handle menu actions
  const handleMenu = (event) => {
    setAnchorEl(event.currentTarget);
    setIsMenuOpen(true);
  };

  const handleClose = () => {
    setIsMenuOpen(false);
  };

  const handleOpenModal = () => {
    setSelectedValue(
      ["REUTERS", "API"].includes(feedTypeChange?.value) ? RadioButtonLabel[2] : RadioButtonLabel[0]
    );
    setOpenModal(true);
  };

  const handleOnChangeValue = (event) => {
    setSelectedValue(event.target.value);
  };

  const handleProceedClick = () => {
    setIsModalOpen(true);
  };

  const handleCloseModal = () => {
    setOpenModal(false);
    setIsModalOpen(false);
    setIsOpen(false);
  };

  // Handle page change
  const handlePageChange = useCallback(
    (newPage) => {
      setPage(newPage);
    },
    [setPage]
  );

  useEffect(() => {
    const newPage = parseInt(searchParams.get("page")) || 1;
    setPage(newPage);
  }, [searchParams]);

  const handleAddNewRateClick = () => {
    navigate(`${paths?.forex?.main}/${paths?.forex?.timeBasedAddNewRate}`, {
      state: { value: "offer" },
    });
  };

  const handleBulkClick = () => {
    navigate(`${paths?.forex?.main}/${paths?.forex?.offerRateConfiguration}`, {
      state: { fromTab: source },
    });
  };
  // Fetch listing data with useCallback
  const fetchData = useCallback(
    (cancelTokenSource) => {
      const formattedSearchValue = /^[0-9]+$/.test(debouncerValue)
        ? `${debouncerValue}.00`
        : debouncerValue;
      const schemeFilter =
        dropdownState.scheme?.length > 0 ? dropdownState.scheme.map((s) => s.value) : [];

      const validFrom = startDate ? startDate.format("YYYY-MM-DDTHH:mm:ss[Z]") : null;
      const validTo = endDate ? endDate.format("YYYY-MM-DDTHH:mm:ss[Z]") : null;
      setLoading(true);
      const destCountry =
        Array.isArray(destCountryCode) && destCountryCode.length > 0
          ? destCountryCode.map((country) => country.value)
          : destCountryCode?.value
            ? [destCountryCode.value]
            : [];
      const srcCurrencyCode =
        Array.isArray(sourceCurrency) && sourceCurrency.length > 0
          ? sourceCurrency.map((currency) => currency.value)
          : sourceCurrency?.value
            ? [sourceCurrency.value]
            : [];
      const destCurrencyCode =
        Array.isArray(destinationCurrency) && destinationCurrency.length > 0
          ? destinationCurrency.map((currency) => currency.value)
          : destinationCurrency?.value
            ? [destinationCurrency.value]
            : [];
      const sendPartnerIdCode =
        Array.isArray(sendPartnerId) && sendPartnerId.length > 0
          ? sendPartnerId.map((partner) =>
            partner.value === "*" ? Encrypt(partner.value) : partner.value
          )
          : sendPartnerId?.value
            ? [sendPartnerId.value === "*" ? Encrypt(sendPartnerId.value) : sendPartnerId.value]
            : [];
      const requestData = {
        ...(srcCurrencyCode.length > 0 && { sellCurrency: srcCurrencyCode }),
        ...(destCurrencyCode.length > 0 && { buyCurrency: destCurrencyCode }),
        ...(destCountry.length > 0 && { destCountryCode: destCountry }),
        ...(sendPartnerIdCode.length > 0 && { partnerId: sendPartnerIdCode }),
        ...(isDateTimeSelected && validFrom && validTo && { validFrom, validTo }),
        ...(schemeFilter.length > 0 && { scheme: schemeFilter }),
        ...{
          fxfeed: dropdownState.fxFeed?.length
            ? dropdownState.fxFeed.map((e) => e.value)
            : ForexData,
        },
        ...(dropdownState.modifiedBy?.length && {
          modifiedBy: dropdownState.modifiedBy
            .filter((e) => e.value)
            .map((e) => encodeURIComponent(e.value)),
        }),
        ...(timeBaseStatus?.length && {
          status:
            timeBaseStatus.value === "all"
              ? statusType?.filter((v) => v.value !== "all")?.map((v) => v?.value)
              : timeBaseStatus?.map((e) => e.value),
        }),
        ...(dropdownState.feedType?.length && {
          feedType: dropdownState.feedType.map((e) => e.value),
        }),
        fxName: TimeBased,
        ...(timeBaseIntrType?.length && {
          instrumentType:
            timeBaseIntrType.length === instrumentTypes.length
              ? [bankKey, walletKey, cardKey, "*"].join(",")
              : timeBaseIntrType.some((e) => e.value === bankKey) &&
                timeBaseIntrType.some((e) => e.value === walletKey) &&
                timeBaseIntrType.length === 2
                ? "*"
                : timeBaseIntrType.map((e) => e.value).join(","),
        }),
        // ...(txnType?.length && {
        //   transactionType:
        //     txnType.length === transactionTypes.length
        //       ? "*"
        //       : txnType?.map((e) => e.value)?.join(","),
        // }),
        ...(txnType?.length && {
          transactionType: txnType.map((e) => e.value).join(","),
        }),
        ...(dropdownState.createdBy?.length && {
          createdBy: dropdownState.createdBy
            .filter((e) => e.value)
            .map((e) => encodeURIComponent(e.value)),
        }),
      };
      if (debouncerValue && debouncerValue.length > 0) requestData.search = formattedSearchValue;

      getTimeBase(requestData, page, limit, orderBy, sortBy, cancelTokenSource?.token)
        .then((response) => {
          const modifiedData = response?.data?.data?.data.map((item, index) => ({
            ...item,
            slno: (page - 1) * limit + index + 1,
          }));
          setTimeBaseData(modifiedData);
          setIsDataAvailable(response?.data?.data?.count > 0);
          setTotalRowsCount(response.data.data.count);
          // setLoading(false);
        })
        .catch((error) => {
          setTimeBaseData([]);
          setIsDataAvailable(false);
          displayToast(error.response.data?.message);
          setLoading(false);
        })
        .finally(() => setTimeout(() => setLoading(false), 1000));
    },
    [
      sourceCurrency,
      destinationCurrency,
      destCountryCode,
      sendPartnerId,
      debouncerValue,
      selectDate,
      dropdownState.fxFeed,
      orderBy,
      sortBy,
      page,
      limit,
      selectedRange,
      dropdownState.modifiedBy,
      dropdownState.createdBy,
      timeBaseStatus,
      dropdownState.feedType,
      dropdownState.scheme,
      timeBaseIntrType,
      txnType,
      startDate,
      endDate,
    ]
  );

  // fetch the rate by id
  const fetchDataById = (ID) => {
    setIsModalLoading(true);
    setIsRateByIdFailed(false);
    const requestData = {
      id: ID,
      fxName: TimeBased,
    };
    getOfferRateById(requestData)
      .then((response) => {
        const responseData = response.data?.data?.data ? response.data.data.data?.[0] : null;
        setRowDataToUpdate(responseData);
        setIsModalLoading(false);
        setIsRateByIdFailed(false);
      })
      .catch((error) => {
        setIsDataAvailable(false);
        displayToast(error.response.data.message);
        setIsModalLoading(false);
        setIsRateByIdFailed(true);
      });
  };
  // Fetch partner dropdown options
  useEffect(() => {
    getPartnerDropDown({ partnerType: "send", fxName: TimeBased }).then((res) => {
      if (res.data.isSuccess === true) {
        setPartnerList(res.data.data);
      }
    });
  }, []);

  const firstRun = useRef(true);

  useEffect(() => {
    const cts = axios.CancelToken.source();
    if (firstRun.current) {
      if (timeBaseStatus?.length > 0) {
        firstRun.current = false;
        fetchData(cts);
      }
    } else {
      fetchData(cts);
    }

    return () => cts.cancel();
  }, [fetchData, sendPartnerId, timeBaseStatus]);

  // Reset page to 1 when dependencies change
  useEffect(() => {
    setPage(1);
  }, [
    sourceCurrency,
    destinationCurrency,
    destCountryCode,
    sendPartnerId,
    debouncerValue,
    selectDate,
    // fxFeed,
    dropdownState.fxFeed,
    dropdownState.modifiedBy,
    limit,
    selectedRange,
  ]);

  // Handle search input change
  const handleSearchChange = (event) => {
    const inputValue = event.target.value.trimStart();

    if (numericSearchFieldValid(inputValue)) {
      setTimeBaseSearch(inputValue);
      searchParams.set("page", 1);
      setSearchParams(searchParams, { replace: true });
    } else {
      displayToast(ERROR_MESSAGES.SEARCH_FIELD_MESSAGE);
    }
  };

  // Handle row click
  const handleRowClick = (rowData) => {
    setSelectedRowData(rowData);
  };

  // Update destination country selection
  const handleCountryChange = (e, value) => {
    setdestCountryCode(value || []);
  };

  // Update destination currency selection
  const handleDestCurrencyChange = (e, value) => {
    setDestinationCurrency(value || []);
  };

  // Update source currency selection
  const handleSrcCurrencyChange = (e, value) => {
    setSourceCurrency(value || []);
  };

  // Update partner selection
  const handlePartnerChange = (e, value) => {
    setSendPartnerId(value || []);
  };

  // Update page limit
  const handleLimitChange = (e) => {
    setLimit(e.target.value);
    searchParams.set("page", 1); // Set page number to 1
    setSearchParams(searchParams);
  };

  const handleCountryModal = () => {
    setViewCountry(false);
  };

  // Handle download details
  const handleDownloadClick = () => {
    const formattedSearchValue = /^[0-9]+$/.test(debouncerValue)
      ? `${debouncerValue}.00`
      : debouncerValue;
    // const validFrom = selectedRange.startDate
    //   ? format(new Date(selectedRange.startDate), DATE_FORMAT)
    //   : null;
    // const validTo = selectedRange.endDate
    //   ? format(new Date(selectedRange.endDate), DATE_FORMAT)
    //   : null;
    const validFrom = startDate ? startDate.format("YYYY-MM-DDTHH:mm:ss[Z]") : null;
    const validTo = endDate ? endDate.format("YYYY-MM-DDTHH:mm:ss[Z]") : null;
    const destCountry =
      Array.isArray(destCountryCode) && destCountryCode.length > 0
        ? destCountryCode.map((country) => country.value)
        : destCountryCode?.value
          ? [destCountryCode.value]
          : [];
    const srcCurrencyCode =
      Array.isArray(sourceCurrency) && sourceCurrency.length > 0
        ? sourceCurrency.map((currency) => currency.value)
        : sourceCurrency?.value
          ? [sourceCurrency.value]
          : [];
    const destCurrencyCode =
      Array.isArray(destinationCurrency) && destinationCurrency.length > 0
        ? destinationCurrency.map((currency) => currency.value)
        : destinationCurrency?.value
          ? [destinationCurrency.value]
          : [];
    const sendPartnerIdCode =
      Array.isArray(sendPartnerId) && sendPartnerId.length > 0
        ? sendPartnerId.map((partner) =>
          partner.value === "*" ? Encrypt(partner.value) : partner.value
        )
        : sendPartnerId?.value
          ? [sendPartnerId.value === "*" ? Encrypt(sendPartnerId.value) : sendPartnerId.value]
          : [];
    const schemeFilter =
      dropdownState.scheme?.length > 0 ? dropdownState.scheme.map((s) => s.value) : [];
    const requestData = {
      ...(srcCurrencyCode.length > 0 && { sellCurrency: srcCurrencyCode }),
      ...(destCurrencyCode.length > 0 && { buyCurrency: destCurrencyCode }),
      ...(destCountry.length > 0 && { destCountryCode: destCountry }),
      ...(sendPartnerIdCode.length > 0 && { partnerId: sendPartnerIdCode }),
      ...(isDateTimeSelected && validFrom && validTo && { validFrom, validTo }),
      ...(formattedSearchValue.length > 0 && { search: formattedSearchValue }),
      ...(schemeFilter.length > 0 && { scheme: schemeFilter }),
      ...{
        fxfeed: dropdownState.fxFeed?.length ? dropdownState.fxFeed.map((e) => e.value) : ForexData,
      },
      ...(dropdownState.modifiedBy?.length && {
        modifiedBy: dropdownState.modifiedBy
          .filter((e) => e.value)
          .map((e) => encodeURIComponent(e.value)),
      }),
      ...(dropdownState.createdBy?.length && {
        createdBy: dropdownState.createdBy
          .filter((e) => e.value)
          .map((e) => encodeURIComponent(e.value)),
      }),
      ...(timeBaseStatus?.value && { status: [timeBaseStatus.value] }),
      ...(feedTypeChange?.value && { feedType: [feedTypeChange.value] }),
      fxName: TimeBased,
      ...(txnType?.length && {
        transactionType: txnType?.map((e) => e.value)?.join(","),
      }),
      ...(timeBaseIntrType?.length && {
        instrumentType:
          timeBaseIntrType.length === instrumentTypes.length
            ? [bankKey, walletKey, cardKey, "*"].join(",")
            : timeBaseIntrType.some((e) => e.value === bankKey) &&
              timeBaseIntrType.some((e) => e.value === walletKey) &&
              timeBaseIntrType.length === 2
              ? "*"
              : timeBaseIntrType.map((e) => e.value).join(","),
      }),
      ...(timeBaseStatus?.length && {
        status:
          timeBaseStatus.value === "all"
            ? statusType?.filter((v) => v.value !== "all")?.map((v) => v?.value)
            : timeBaseStatus?.map((e) => e.value),
      }),
      ...(dropdownState.feedType?.length && {
        feedType: dropdownState.feedType.map((e) => e.value),
      }),
    };
    const apiSortBy = sortBy === "src_partner_id" ? "partner_name" : sortBy;
    requestData.sortBy = apiSortBy;
    timeBaseDownload(requestData, orderBy, apiSortBy)
      .then((response) => {
        const tableName = TimeBaseBulkTable;
        downloadData(response, tableName); // Call the download function to trigger download
        setIsOpen(false);
      })
      .catch((error) => {
        setIsOpen(false);
        displayToast(error?.response?.data?.message); // Error Message Download
      });
  };

  const handlePartnerCloseModal = () => {
    setViewPartners(false);
  };

  const openPartnerModal = (fxRateId) => {
    setIsLoading(true);

    timeBaseDefaultPartner(fxRateId)
      .then((res) => {
        if (res.data.isSuccess === true) {
          const decryptedPartnersList = res.data.data
            .map((partner) => ({
              ...partner,
              partner_id: Decrypt(partner.partner_id),
            }))
            .filter((partner) => partner.partner_txn_type !== "Receiver")
            .sort((a, b) => a.partner_name.localeCompare(b.partner_name));
          setPartnersList(decryptedPartnersList);
          setViewPartners(true);
        }
      })
      .catch((error) => {
        displayToast(error?.response?.data?.message);
      })
      .finally(() => {
        setIsLoading(false);
      });
  };

  const openCountryModal = (dest_currency) => {
    setIsLoading(true);
    countryModalList(dest_currency)
      .then((res) => {
        setCountryListVal(
          res.data.data.list.sort((a, b) => a.country_name.localeCompare(b.country_name))
        );
        setViewCountry(true);
      })
      .catch((error) => {
        displayToast(error.response.data.message);
      })
      .finally(() => {
        setIsLoading(false);
      });
  };

  // Additional state variables for menu and recall modal
  const [menuAnchorEl, setMenuAnchorEl] = useState(null);
  const [isRecallModalOpen, setIsRecallModalOpen] = useState(false);
  const [recallModalMessage, setRecallModalMessage] = useState("");
  const [recallComment, setRecallComment] = useState("");
  const [isRecallProcessing, setIsRecallProcessing] = useState(false);

  useEffect(() => {
    getAllUserAccounts()
      .then((userAccountRes) => {
        let userAccountData = [];
        if (userAccountRes?.data?.success) {
          const data = userAccountRes?.data?.data;
          const sortedData = data
            .filter((item) => item?.account_id)
            .sort((a, b) =>
              a?.firstname.replace(/\s/g, "").localeCompare(b?.firstname.replace(/\s/g, ""), "en", {
                sensitivity: "base",
              })
            );
          userAccountData = sortedData.map((option) => ({
            value: option?.account_id,
            label: option?.firstname,
          }));
        }

        getFxFeedName()
          .then((feedNameRes) => {
            let feedNameData = [];
            if (feedNameRes?.data?.data) {
              feedNameData = feedNameRes?.data?.data.map((feedName) => ({
                value: Encrypt(feedName),
                label: feedName,
              }));
              feedNameData.sort((a, b) => a.label.localeCompare(b.label));
            }
            const combinedData = [...userAccountData, ...feedNameData];

            setUserList(combinedData);
          })
          .catch((err) => {
            displayToast(err?.response?.data?.message);
          });
      })
      .catch((err) => {
        displayToast(err?.response?.data?.message);
      });
  }, []);
  // Function to get the current local time
  const getCurrentLocalTime = () => new Date();

  // Function to calculate the difference in hours and minutes between expiryDate and currentTime
  const calculateHourDifference = (expiryDate, currentTime) => {
    const diff = expiryDate - currentTime;
    const hours = Math.floor(diff / 1000 / 60 / 60);
    const minutes = Math.floor((diff % (1000 * 60 * 60)) / (1000 * 60));
    return { hours, minutes };
  };
  const handleUpdateClick = (rowData) => {
    setRowDataToUpdate(selectedRowData);
    fetchDataById(rowData?.id);
    handleOpenModal();
  };

  const mappedSchemesList = useMemo(() => {
    return schemesList.map((s) => ({
      label: s.fullName,
      value: s.label,
    }));
    //  .sort((a, b) => a.label.localeCompare(b.label));
  }, [schemesList]);

  // TimeBase offer rate table columns definition
  const cols = [
    {
      field: "slno",
      headerName: "SL No.",
      minWidth: 80,
      maxWidth: 110,
      sortable: false,
      flex: 2,
      renderCell: (cellValues) => {
        return <div>{cellValues?.value ? cellValues?.value : "-"}</div>;
      },
    },
    {
      field: "src_currency",
      headerName: "Src. Currency",
      minWidth: 100,
      sortable: true,
      flex: 2,
      renderCell: (cellValues) => {
        const valid_to = cellValues?.row?.valid_to ?? "";
        const sourceCurrencyValue = cellValues?.value ? cellValues?.value : "-";
        const sourceCurrencyFullName =
          cellValues?.row?.srcCurrencyName?.currency_abbreviation ?? "N/A";

        // If no valid_to date is provided, display the source currency with a tooltip
        if (!valid_to) {
          return <Tooltip title={sourceCurrencyFullName}>{sourceCurrencyValue}</Tooltip>;
        }

        const expiryDate = new Date(valid_to);
        const currentTime = getCurrentGmtTime();
        const { hours, minutes } = calculateHourDifference(expiryDate, currentTime);

        // Display expired status if the offer has expired
        if (hours <= 0 && minutes <= 0) {
          return (
            <div>
              <div>
                <Tooltip title={sourceCurrencyFullName}>{sourceCurrencyValue}</Tooltip>
              </div>
              <span
                style={{
                  fontSize: "8px",
                  backgroundColor: Colors.tpRed,
                  borderRadius: "2px",
                  color: Colors.light,
                  whiteSpace: "pre",
                  padding: "1px",
                }}
              >
                &nbsp;Expired&nbsp;
              </span>
            </div>
          );
        }

        // Return the source currency value with a tooltip
        return (
          <div>
            <Tooltip title={sourceCurrencyFullName}>{sourceCurrencyValue}</Tooltip>
          </div>
        );
      },
    },
    {
      field: "dest_currency",
      headerName: "Dest. Currency",
      minWidth: 100,
      sortable: true,
      flex: 2,
      renderCell: (cellValues) => {
        const destinationCurrencyValue = cellValues?.value ? cellValues?.value : "-";
        const destinationCurrencyFullName =
          cellValues?.row?.destCurrencyName?.currency_abbreviation ?? "N/A";
        return (
          <Tooltip title={destinationCurrencyFullName}>
            <div>{destinationCurrencyValue}</div>
          </Tooltip>
        );
      },
    },
    {
      field: "dest_country_code",
      headerName: "Dest. Country",
      minWidth: 100,
      sortable: true,
      flex: 2,
      renderCell: (cellValues) => {
        const countryName = cellValues.row.CountryName?.country_code || "-";
        const fullCountryName = cellValues.row.CountryName?.country_name || "-";
        const countryCount = cellValues.row.countryCount;
        const srcCurrency = cellValues.row.src_currency;
        const dest_currency = cellValues.row.dest_currency;

        // Handle click to view country modal
        const handleViewCountryClick = () => {
          if (cellValues?.value === "*") {
            openCountryModal(dest_currency);
          }
        };

        // Display default if country count is zero
        if (countryCount === 0) {
          return (
            <div style={{ textAlign: "center" }}>
              <span>Default</span>
            </div>
          );
        } else if (cellValues?.value === "*") {
          return (
            <span
              style={{
                color: Colors.tpBlue,
                cursor: "pointer",
                textDecoration: "underline",
              }}
              onClick={handleViewCountryClick}
            >
              Default
            </span>
          );
        } else {
          return (
            <div
              style={{
                margin: cellValues.row.CountryName?.country_name ? "left" : "auto",
              }}
            >
              <Tooltip title={cellValues.row.CountryName?.country_name}>
                <div>{countryName ? countryName : "-"}</div>
              </Tooltip>
            </div>
          );
        }
      },
    },
    {
      field: "src_partner_id",
      headerName: "Partner",
      minWidth: 100,
      flex: 1,
      sortable: true,
      renderCell: (cellValues) => {
        const partnerCount = cellValues.row.partnerCount;
        const encryptedPartnerId = cellValues.row.src_partner_id;
        const decryptedPartnerId = Decrypt(encryptedPartnerId);
        const countryCode = cellValues.row?.countryList || "-";
        const fxRateId = cellValues.row.id;

        // Handle click to view partner modal
        const handleViewPartnerClick = () => {
          if (decryptedPartnerId === "*") {
            const countryCodes = countryCode;
            openPartnerModal(fxRateId, countryCodes);
          }
        };

        // Display default if partner count is zero
        if (partnerCount === 0) {
          return (
            <div style={{ textAlign: "center" }}>
              <span>Default</span>
            </div>
          );
        } else if (decryptedPartnerId === "*") {
          return (
            <div>
              <span
                style={{ cursor: "pointer", textDecoration: "underline" }}
                onClick={handleViewPartnerClick}
              >
                Default
              </span>
            </div>
          );
        } else {
          const partnerName = cellValues.row.partnerName?.partner_name || "-";
          const truncatedName =
            partnerName.length > 20 ? partnerName.slice(0, 20) + "..." : partnerName;

          return (
            <div style={{ textAlign: cellValues?.value ? "left" : "center" }}>
              <Tooltip title={partnerName} placement="bottom-start">
                <div
                  style={{
                    color: Colors.tpBlue,
                    overflow: "hidden",
                    whiteSpace: "nowrap",
                    textOverflow: "ellipsis",
                    cursor: "pointer",
                    width: "120px",
                  }}
                >
                  {truncatedName}
                </div>
              </Tooltip>
            </div>
          );
        }
      },
    },
    // {
    //   field: "fx_feed",
    //   headerName: (
    //     <>
    //       <Box mt={1} sx={{ overflow: "visible" }}>
    //         <MultiSelectDropdown
    //           label={"Forex Type"}
    //           // backgroundColor={`${Colors.tpLightBlue}`}
    //           width={"145px"}
    //           chipWidth={"80%"}
    //           selectedOption={dropdownState.fxFeed || fxFeed}
    //           options={forexType}
    //           showTooltip={false}
    //           onChange={(e, details) => {
    //             // setFxFeed(details);
    //             setDropdownState((prev) => ({ ...prev, fxFeed: details }));
    //             setPage(1);
    //             searchParams.set("page", 1);
    //             setSearchParams(searchParams, { replace: true });
    //           }}
    //           {...commonMultiSelectProps}
    //         />
    //       </Box>
    //     </>
    //   ),
    //   minWidth: 170,
    //   flex: 1,
    //   sortable: false,
    //   renderCell: (cellValues) => {
    //     return (
    //       <div>
    //         <div>{cellValues?.value ? cellValues?.value : "-"}</div>
    //       </div>
    //     );
    //   },
    // },
    {
      field: "offered_bid_rate",
      headerName: "Offered Rate Bid",
      minWidth: 120,
      headerAlign: "center",
      align: "right",
      flex: 1,
      type: "number",
      sortable: true,
      renderCell: (cellValues) => {
        return (
          <>
            <Tooltip title={cellValues?.value ? cellValues?.value : "-"} placement="bottom">
              <div
                style={{
                  whiteSpace: "nowrap",
                  overflow: "hidden",
                  textOverflow: "ellipsis",
                  maxWidth: "100px",
                }}
              >
                {cellValues?.value ? cellValues?.value : "-"}
              </div>
            </Tooltip>
            <br />
          </>
        );
      },
    },
    {
      field: "offered_ask_rate",
      headerName: "Offered Rate Ask",
      minWidth: 120,
      headerAlign: "center",
      align: "right",
      flex: 1,
      sortable: true,
      renderCell: (cellValues) => {
        return (
          <>
            <Tooltip title={cellValues?.value ? cellValues?.value : "-"} placement="bottom">
              <div
                style={{
                  whiteSpace: "nowrap",
                  overflow: "hidden",
                  textOverflow: "ellipsis",
                  maxWidth: "100px",
                }}
              >
                {cellValues?.value ? cellValues?.value : "-"}
              </div>
            </Tooltip>
          </>
        );
      },
    },
    {
      field: "valid_from",
      headerName: "Validity From (GMT)",
      minWidth: 150,
      sortable: true,
      renderCell: (cellValues) => {
        const formattedDateTime = DateTimeFormat(cellValues?.value, false);
        return (
          <div style={{ whiteSpace: "nowrap" }}>
            <span>{formattedDateTime.dateAndTime}</span>
          </div>
        );
      },
    },
    {
      field: "valid_to",
      headerName: "Validity To (GMT)",
      minWidth: 150,
      sortable: true,
      renderCell: (cellValues) => {
        const formattedDateTime = DateTimeFormat(cellValues?.value, false);
        const expiryDate = new Date(cellValues?.value);
        const currentTime = getCurrentGmtTime();
        const { hours } = calculateHourDifference(expiryDate, currentTime);

        const isNearingExpiry = hours < RATE_EXPIRATION_TIME && hours >= 0;

        return (
          <div
            style={{
              whiteSpace: "nowrap",
              display: "flex",
              flexDirection: "column",
            }}
          >
            <span>{formattedDateTime.dateAndTime}</span>
            {isNearingExpiry && (
              <span
                style={{
                  fontSize: "8px",
                  backgroundColor: "lightgreen",
                  borderRadius: "2px",
                  color: Colors.light,
                  whiteSpace: "pre",
                  padding: "2px 4px",
                  marginTop: "4px",
                  display: "inline-block",
                  textAlign: "center",
                  width: "max-content",
                }}
              >
                Nearing Expiry
              </span>
            )}
          </div>
        );
      },
    },

    {
      field: "scheme",
      headerName: (
        <Box mt={1} sx={{ minWidth: 170 }}>
          <MultiSelectDropdown
            label="Schemes"
            width={"150px"}
            selectedOption={dropdownState.scheme || []}
            options={mappedSchemesList}
            onChange={(e, details) => {
              setDropdownState((prev) => ({ ...prev, scheme: details }));
              setPage(1);
              searchParams.set("page", 1);
              setSearchParams(searchParams, { replace: true });
            }}
            {...commonMultiSelectProps}
            showSelectAllOption={true}
          />
        </Box>
      ),
      width: 190,
      minWidth: 190,
      flex: 1,
      sortable: false,
      renderCell: (cellValues) => {
        let scheme;

        if (cellValues.value === "*") {
          scheme = { fullName: DEFAULT_TEXT };
        } else {
          scheme = schemesList.find((t) => t.label === cellValues.value);
        }
        return (
          <div style={{ overflowWrap: "anywhere" }}>
            <span>{scheme ? scheme.fullName : "-"}</span>
          </div>
        );
      },
    },
    {
      field: "feed_type",
      headerName: (
        <>
          <Box mt={1}>
            <MultiSelectDropdown
              label={"Feed Type"}
              width={"160px"}
              selectedOption={dropdownState.feedType}
              options={feedTypeOpt || []}
              onChange={(e, details) => {
                setDropdownState((prev) => ({ ...prev, feedType: details }));
                setPage(1);
                searchParams.set("page", 1);
                setSearchParams(searchParams, { replace: true });
              }}
              {...commonMultiSelectProps}
            />
          </Box>
        </>
      ),
      width: 190,
      sortable: false,
      renderCell: (cellValues) => {
        return (
          <div style={{ overflowWrap: "anywhere" }}>
            <span>{cellValues?.value ? cellValues?.value : "-"}</span>
          </div>
        );
      },
    },
    {
      field: "instrument_type",
      headerName: (
        <>
          <Box mt={1}>
            <MultiSelectDropdown
              label={"Instr. Type"}
              width={"155px"}
              options={instrumentTypes}
              showTooltip={false}
              selectedOption={timeBaseIntrType ? timeBaseIntrType : null}
              onChange={(e, details) => {
                setTimeBaseInstrType(details);
                setPage(1);
                searchParams.set("page", 1);
                setSearchParams(searchParams, { replace: true });
              }}
              {...commonMultiSelectProps}
            />
          </Box>
        </>
      ),
      width: 180,
      minWidth: 180,
      flex: 1,
      sortable: false,
      renderCell: (cellValues) => {
        const value =
          cellValues?.value === "*"
            ? sourceRateMessage.FX_BANK_WALLET.join(" & ")
            : instrumentTypes?.find(
              (v) => v.value?.toUpperCase() === cellValues?.value?.toUpperCase()
            )?.label || "-";

        return <span style={{ overflowWrap: "anywhere" }}>{value}</span>;
      },
    },
    {
      field: "transaction_type",
      headerName: (
        <>
          <Box mt={1}>
            <MultiSelectDropdown
              label={"Txn. Type"}
              width={"155px"}
              selectedOption={txnType ? txnType : null}
              options={transactionTypes}
              onChange={(e, details) => {
                setTxnType(details);
                setPage(1);
                searchParams.set("page", 1);
                setSearchParams(searchParams, { replace: true });
              }}
              {...commonMultiSelectProps}
            // showSelectAllOption={false}
            />
          </Box>
        </>
      ),
      width: 180,
      sortable: false,
      renderCell: (cellValues) => {
        const transactionTypeDisplay = cellValues?.value === "*" ? DEFAULT_TEXT : cellValues?.value;
        return (
          <div style={{ overflowWrap: "anywhere" }}>
            <span>{transactionTypeDisplay || "-"}</span>
          </div>
        );
      },
    },

    {
      field: "status",
      headerName: (
        <>
          <Box mt={1}>
            <MultiSelectDropdown
              label={"Status"}
              width={"160px"}
              selectedOption={timeBaseStatus}
              options={statusOpt}
              onChange={(e, details) => {
                setTimeBaseStatus(details);
                setPage(1);
                searchParams.set("page", 1);
                setSearchParams(searchParams, { replace: true });
              }}
              {...commonMultiSelectProps}
            />
          </Box>
        </>
      ),
      width: 190,
      sortable: false,
      renderCell: (cellValues) => {
        const statusColor = getStatusColor(cellValues?.value);
        return (
          <div style={{ overflowWrap: "anywhere", color: statusColor }}>
            <span>{cellValues?.value ? cellValues?.value : "-"}</span>
          </div>
        );
      },
    },
    {
      field: "approved_on",
      headerName: "Approved On",
      width: 150,
      sortable: true,
      renderCell: (cellValues) => {
        if (!cellValues?.value) {
          return <span>-</span>;
        }
        const formattedApprovedOn = DateTimeFormat(cellValues?.value);

        return (
          <div style={{ overflowWrap: "anywhere" }}>
            <span>{formattedApprovedOn.dateAndTime}</span>
          </div>
        );
      },
    },
    {
      field: "approvedBy",
      headerName: "Approved By",
      width: 190,
      sortable: false,
      // flex: 1,
      renderCell: (cellValues) => {
        // const firstname = cellValues.row["approvedBy.firstname"] || "-";
        return (
          <div style={{ overflowWrap: "anywhere" }}>
            <div>{cellValues?.value ? cellValues?.value : "-"}</div>
          </div>
        );
      },
    },
    {
      field: "modified_on",
      headerName: "Modified On",
      width: 150,
      sortable: true,
      renderCell: (params) => {
        if (!params.row.created_on && !params.value) {
          return <span>-</span>;
        }
        const created = params.row.created_on ? DateTimeFormat(params.row.created_on) : null;
        const modified = params.value ? DateTimeFormat(params.value) : null;
        const display = modified || created;
        return (
          <div style={{ overflowWrap: "anywhere" }}>
            <span>{display.dateAndTime}</span>
          </div>
        );
      },
    },
    {
      field: "modified_by",
      headerName: (
        <>
          <Box mt={1}>
            <MultiSelectDropdown
              label={"Modified By"}
              width={"170px"}
              selectedOption={dropdownState.modifiedBy || null}
              options={userList || []}
              onChange={(e, details) => {
                setDropdownState((prev) => ({ ...prev, modifiedBy: details }));
                setPage(1);
                searchParams.set("page", 1);
                setSearchParams(searchParams, { replace: true });
              }}
              {...commonMultiSelectProps}
              showTooltip={false}
            />
          </Box>
        </>
      ),
      width: 190,
      sortable: false,
      renderCell: (cellValues) => {
        const createdBy = cellValues.row.createdBy;
        const modifiedBy = cellValues.row.modifiedBy;
        const userName = modifiedBy || createdBy || "-";
        return (
          <div style={{ overflowWrap: "anywhere" }}>
            <div>{`${userName}`}</div>
          </div>
        );
      },
    },
  ];

  // If the user has edit access, add the update column
  if (permissions.hasEditAccess) {
    cols.push({
      field: "linkupdate",
      headerName: "",
      width: 100,
      sortable: false,
      renderCell: (cellValues) => {
        const currentDate = getCurrentGmtTime();
        const validToDate = new Date(cellValues.row.valid_to);
        const isActiveOrPending = [active, DealsMsgs.hold].includes(cellValues.row.status);

        if (!isActiveOrPending || validToDate.getTime() < currentDate) {
          return (
            <div>
              <span
                style={{
                  color: "#999",
                  textDecoration: "underline",
                  cursor: "not-allowed",
                }}
                disabled
              >
                Update
              </span>
            </div>
          );
        } else {
          return (
            <div>
              <span
                style={{
                  color: Colors.tpBlue,
                  textDecoration: "underline",
                  cursor: "pointer",
                }}
                onClick={() => handleUpdateClick(cellValues.row)}
              >
                Update
              </span>
            </div>
          );
        }
      },
    });
  }

  // State for menu anchor
  const [anchorE2, setAnchorE2] = useState(null);

  const handleClick = (event) => {
    setAnchorE2(event.currentTarget);
  };

  const handleClose2 = () => {
    setAnchorE2(null);
  };

  const open = Boolean(anchorE2);
  const id = open ? "simple-popover" : undefined;

  // Dropdown configurations for filters
  const dropdowns = [
    {
      selectedOption: sourceCurrency,
      onChange: handleSrcCurrencyChange,
      options: currency,
      label: "Src. Currency",
    },
    {
      selectedOption: destinationCurrency,
      onChange: handleDestCurrencyChange,
      options: currency,
      label: "Dest. Currency",
    },
    {
      selectedOption: destCountryCode,
      onChange: handleCountryChange,
      options: country,
      defaultOption: DEFAULT_OPTION_COUNTRY,
      label: "Dest. Country",
    },
    {
      selectedOption: sendPartnerId,
      onChange: handlePartnerChange,
      options: Partner,
      defaultOption: DEFAULT_OPTION_PARTNER,
      label: "Partner",
    },
  ];

  // Log data availability
  useEffect(() => {
    console.log("isDataAvailable:", isDataAvailable);
  }, [isDataAvailable]);

  // Partner table headers
  const partnerHeaders = [
    { label: "SI No.", field: "siNo", color: Colors.tpBlue },
    { label: "Partner ID", field: "partner_id", color: Colors.tpBlue },
    { label: "Partner Name", field: "partner_name", color: Colors.tpBlue },
    { label: "Partner Type", field: "partner_txn_type", color: Colors.tpBlue },
  ];

  // Add index to each partner in the list
  const partnerDataWithIndex = partnersList.map((partner, index) => ({
    siNo: index + 1,
    ...partner,
  }));

  // Country table headers
  const countryHeaders = [
    { label: "SI No.", field: "siNo", color: Colors.tpBlue },
    { label: "Country Name", field: "country_name", color: Colors.tpBlue },
  ];

  // Add index to each country in the list
  const countryDataWithIndex = countryListVal.map((country, index) => ({
    siNo: index + 1,
    ...country,
  }));
  const handleDeleteIconClick = () => {
    if (selectedRows.length > 0) {
      setIsDeletePopupOpen(true);
    }
  };

  const handleCancelDelete = () => {
    setIsDeletePopupOpen(false);
  };
  // bulk delete
  const onClickDeleteAllHandler = () => {
    setIsProcessing(true);
    setDeleteAll((prev) => ({ ...prev, loading: true }));
    const requestBody = {
      ids: selectedRows,
      type: "deleteAllCurrentRate",
    };
    updateTimeBaseRate(requestBody)
      .then(() => {
        setDeleteAll((prev) => ({ ...prev, modal: true }));
        setIsDeletePopupOpen(false);
      })
      .catch((error) => {
        setIsProcessing(false);
        displayToast(error?.response?.data?.message);
      })
      .finally(() => {
        setDeleteAll((prev) => ({ ...prev, loading: false }));
        setSelectedRows([]);
        setIsProcessing(false);
      });
  };
  // validating for bulk delete
  const isRowSelectable = useMemo(() => {
    return (params) => {
      const { status, valid_to } = params.row;
      const upperStatus = status?.toUpperCase();

      if (upperStatus === "DELETED") return false;
      // if (upperStatus === "ACTIVE") return true;
      if ([active, DealsMsgs.hold, "EXPIRED"].includes(upperStatus)) return true;

      // if (upperStatus === "EXPIRED") {
      //   const currentGmt = moment.utc();
      //   const validToTime = moment.utc(valid_to);
      //   const hoursDiff = currentGmt.diff(validToTime, 'hours');
      //   return hoursDiff <= expHour && hoursDiff >= 0;
      // }

      return false;
    };
  }, [expHour]);

  const defaultStatusSet = useRef(false);
  // bydefault pending approval ,hold & active selected
  useEffect(() => {
    if (
      statusType &&
      statusType.length > 0 &&
      (!timeBaseStatus || timeBaseStatus.length === 0) &&
      !defaultStatusSet.current
    ) {
      const activeStatus = statusType.find((status) => status.value === DEFAULT_STATUS.value);
      const pendingApprovalStatus = statusType.find(
        (status) => status.value === "PENDING_APPROVAL"
      );
      const holdStatus = statusType.find((status) => status.value === "HOLD");

      const defaultStatuses = [];
      if (activeStatus) defaultStatuses.push(activeStatus);
      if (pendingApprovalStatus) defaultStatuses.push(pendingApprovalStatus);
      if (holdStatus) defaultStatuses.push(holdStatus);

      setTimeBaseStatus(defaultStatuses);
      defaultStatusSet.current = true;
    }
  }, [statusType]);
  // Handle date range
  const handleDateRangeReset = () => {
    setStartDateTime(dayjs.utc());
    setEndDateTime(dayjs.utc().add(1, "day"));
    setEndDate(null);
    setIsDateTimeSelected(false);
  };

  return (
    <>
      {isLoading && (
        <Backdrop
          sx={{
            color: Colors.light,
            zIndex: (theme) => theme.zIndex.drawer + 1,
          }}
          open={isLoading}
        >
          <CircularProgress color="inherit" />
        </Backdrop>
      )}

      <Grid2
        container
        sx={{
          alignItems: "end",
          marginTop: "20px",
          marginBottom: "15px",
          padding: "0px 15px",
        }}
      >
        <Grid2 size={8.0}>
          <Grid2 container spacing={{ xs: 1, md: 1 }} sx={{ alignItems: "end" }}>
            {dropdowns.map((dropdown, index) => (
              <Grid2 size={3} key={index}>
                <MultiSelectDropdown
                  selectedOption={dropdown.selectedOption}
                  onChange={dropdown.onChange}
                  options={dropdown.options}
                  defaultOption={dropdown.defaultOption}
                  label={dropdown.label}
                  multiple
                  checkBox={true}
                  size="small"
                  height="40px"
                  chipWidth="100px"
                />
              </Grid2>
            ))}
          </Grid2>
        </Grid2>

        <Grid2 size={4.0}>
          <Box
            sx={{
              display: "flex",
              alignItems: "center",
              justifyContent: "flex-end",
              gap: 2,
              flexWrap: "wrap",
            }}
          >
            <div>
              <Popover
                id={id}
                open={open}
                anchorEl={anchorE2}
                onClose={handleClose2}
                anchorOrigin={{
                  vertical: "bottom",
                  horizontal: "left",
                }}
              >
                <Box sx={{ p: 2 }}>
                  <TextFieldComponent
                    style={{ minWidth: "120px" }}
                    type="text"
                    placeholder="Offer Rate (Bid / Ask)"
                    value={timeBaseSearch}
                    onChange={handleSearchChange}
                    InputProps={{
                      endAdornment: (
                        <InputAdornment position="end">
                          <IconButton>
                            <SearchIcon height={"16px"} width={"16px"} />
                          </IconButton>
                        </InputAdornment>
                      ),
                    }}
                  />
                </Box>
              </Popover>
            </div>
            {permissions.hasCreateAccess && (
              <CommonButton
                onClick={handleAddNewRateClick}
                style={{
                  width: "max-content",
                  maxWidth: "100%",
                  whiteSpace: "nowrap",
                }}
              >
                Add New Rate
              </CommonButton>
            )}
            <ResultPerPageComponent countPerPage={resultPerPage} limit={limit} handleLimitChange={handleLimitChange} />
            <EventIcon style={{ cursor: "pointer" }} onClick={() => setIsDialogOpen(true)} />
            <DateTimeRangePickerDialog
              open={isDialogOpen}
              onClose={() => setIsDialogOpen(false)}
              startDateTime={startDateTime}
              onStartChange={setStartDateTime}
              endDateTime={endDateTime}
              onEndChange={setEndDateTime}
              onSaveDateRange={(from, to) => {
                setStartDate(from);
                setEndDate(to);
                setIsDateTimeSelected(true);
              }}
              onReset={handleDateRangeReset}
            />

            <SearchIcon aria-describedby={id} onClick={handleClick} style={{ cursor: "pointer" }} fontSize="large" />
            <DownloadIcon
              fontSize="large"
              style={{ cursor: isDataAvailable ? "Pointer" : "Not-Allowed" }}
              onClick={() => {
                if (isDataAvailable) {
                  setIsOpen(true);
                }
              }}
            />
            {permissions.hasCreateAccess && <MoreIcon fontSize="large" style={{ cursor: "pointer" }} onClick={handleMenu} />}
          </Box>
        </Grid2>
      </Grid2>

      <Menu
        id="simple-menu"
        anchorEl={anchorEl}
        keepMounted
        open={isMenuOpen}
        onClose={handleClose}
        slotProps={{
          style: {
            boxShadow: "0px 2px 5px rgba(0,0,0,0.3)",
            fontFamily: "Space Grotesk",
            fontSize: "14px",
            color: Colors.tpBlue,
          },
        }}
      >
        <MenuItem onClick={handleBulkClick}>Add Bulk Rate</MenuItem>
      </Menu>

      <Grid2 container>
        <Grid2 size={12}>
          <Box
            sx={{
              float: "right",
              display: "flex",
              paddingRight: "11px",
              gap: "20px",
              paddingBottom: "10px",
            }}
          ></Box>
        </Grid2>
      </Grid2>
      {permissions.hasDeleteAccess && selectedRows?.length > 0 && (
        <Box sx={styles.fadeInAnimation}>
          {selectedRows.length} Rates selected -{" "}
          <span style={{ cursor: "pointer" }} onClick={handleDeleteIconClick}>
            {deleteAll?.loading ? (
              <Box component="span" paddingLeft={"30px"} direction="row" minHeight={"1.991vh"} justifyContent="flex-start" alignItems="center">
                <CircularProgress size={14} />
              </Box>
            ) : (
              <>
                {" "}
                Delete All
                <DeleteIcon style={styles.deleteAllIcon} />{" "}
              </>
            )}
          </span>
        </Box>
      )}
      <Box>
        <div
          style={{
            display: "flex",
            flexDirection: "column",
            justifyContent: "flex-end",
            position: "relative",
            bottom: "10px",
            width: "100%",
          }}
        >
          <div
            style={{
              minHeight: "calc(100% - 10px)",
              width: "100%",
              overflowY: "auto",
              marginTop: "7px",
            }}
          >
            <DataGridCommponent
              data={timebasedata}
              columns={cols}
              getRowId={(r) => r.id}
              customHeight={"auto"}
              rowColor={Colors?.light}
              colors={Colors}
              onRowClick={handleRowClick}
              callback={(a, b, c) => {
                setSortBy(c[0]?.field);
                setOrderBy(c[0]?.sort?.toUpperCase());
                setSortInitialState(c);
              }}
              sortInitialState={sortInitialState}
              checkboxSelection={permissions.hasDeleteAccess}
              rowSelectionModel={selectedRows}
              onRowSelectionModelChange={handleRowSelection}
              loading={loading}
              isRowSelectable={isRowSelectable}
            ></DataGridCommponent>
          </div>
        </div>
        <Box>
          <CustomPaginationComponent
            totalItemsCount={totalRowsCount}
            page={page}
            numberOfItemsPerPage={limit}
            onChangePage={handlePageChange}
            colors={{ tpBlue: Colors.tpBlue }}
          />
        </Box>
      </Box>
      {/* Update rate */}
      {isModalLoading ?
        (<CustomBackdrop mainLoading={isModalLoading} />)
        : (<CommonModal
          open={openModal}
          setOpen={setOpenModal}
          subTitle="Select Update Action"
          titleIcon={<UpdateArrowIcon />}
          width={"30%"}
          height={"auto"}
          fontSize="18px"
        >
          <>
            <CustomRadioButton
              selectedValue={selectedValue}
              labelList={selectedRowData?.status?.toUpperCase() === DealsMsgs?.hold ? [RadioButtonLabel[0], RadioButtonLabel[4]] : RadioButtonLabel}
              defaultValue={RadioButtonLabel[0]}
              mandantory={true}
              handleOnchangeValue={handleOnChangeValue}
              marginLeft="10px"
            />
            <Box style={{ width: "40%", margin: "auto" }} p={1}>
              <CommonButton onClick={handleProceedClick} disabled={isRateByIdFailed}>
                Proceed
              </CommonButton>
            </Box>
            {isModalOpen && selectedValue === RadioButtonLabel[0] && (
              <CommonModal open={setIsModalOpen} selectedValue={selectedValue} handleClose={handleCloseModal} title="Update New Rate" titleIcon={<AddRateIcon />}>
                <AddNewContent
                  handleCloseModal={handleCloseModal}
                  rowData={rowDataToUpdate}
                  fetchData={fetchData}
                  setIsModalOpen={setIsModalOpen}
                  instrumentTypes={instrumentTypes}
                  transactionTypes={transactionTypes}
                />
              </CommonModal>
            )}
            {isModalOpen && selectedValue === RadioButtonLabel[1] && (
              <CommonModal open={setIsModalOpen} selectedValue={selectedValue} handleClose={handleCloseModal} title="Extend Validity" titleIcon={<AddRateIcon />}>
                <ExtendValidity
                  handleCloseModal={handleCloseModal}
                  rowData={rowDataToUpdate}
                  fetchData={fetchData}
                  setIsModalOpen={setIsModalOpen}
                  instrumentTypes={instrumentTypes}
                  transactionTypes={transactionTypes}
                />
              </CommonModal>
            )}
            {isModalOpen && selectedValue === RadioButtonLabel[2] && (
              <CommonModal
                open={setIsModalOpen}
                selectedValue={selectedValue}
                handleClose={handleCloseModal}
                title="Expire Current Rate & Add New Rate"
                titleIcon={<AddRateIcon />}
              >
                <ExtendAndAddNew
                  handleCloseModal={handleCloseModal}
                  rowData={rowDataToUpdate}
                  fetchData={fetchData}
                  setIsModalOpen={setIsModalOpen}
                  instrumentTypes={instrumentTypes}
                  transactionTypes={transactionTypes}
                />
              </CommonModal>
            )}
            {isModalOpen &&
              (selectedValue === RadioButtonLabel[3] || selectedValue === RadioButtonLabel[4]) &&
              (() => {
                const isDelete = selectedValue === RadioButtonLabel[4];
                const title = isDelete ? "Delete Rate" : "Expire Current Rate";

                return (
                  <CommonModal
                    open={setIsModalOpen}
                    selectedValue={selectedValue}
                    handleClose={handleCloseModal}
                    title={title}
                    titleIcon={<AddRateIcon />}
                    width="65%"
                  >
                    <ExtendCurrentRate
                      handleCloseModal={handleCloseModal}
                      setOpenModal={setOpenModal}
                      rowData={rowDataToUpdate}
                      fetchData={fetchData}
                      hasDeleteAccess={permissions.hasDeleteAccess}
                      setIsModalOpen={setIsModalOpen}
                      instrumentTypes={instrumentTypes}
                      {...(isDelete ? { deleteModal: true } : {})}
                    />
                  </CommonModal>
                );
              })()}
          </>
        </CommonModal>)
      }

      <CommonModal open={viewPartners} subTitle="View Partners" handleClose={handlePartnerCloseModal} fontSize="18px" headerPadding="15px 15px" width="auto">
        <Stack sx={{ width: "100%", overflow: "auto", boxShadow: "none" }}>
          <ReusableTable headers={partnerHeaders} data={partnerDataWithIndex} rowKey="partner_id" />
        </Stack>
      </CommonModal>

      <CommonModal open={viewCountry} subTitle="Country List" handleClose={handleCountryModal} width={"500px"} fontSize="18px" headerPadding="15px 15px">
        <Stack sx={{ width: "100%", overflow: "hidden", boxShadow: "none" }}>
          <ReusableTable headers={countryHeaders} data={countryDataWithIndex} rowKey="country_code" />
        </Stack>
      </CommonModal>
      <CommonModal
        open={isOpen}
        handleClose={handleCloseModal}
        title={"Download"}
        width="800px"
        titleIcon={<WhiteDownloadIcon fontSize="large" color="white" />}
        headerPadding="20px 20px"
        fontSize="12px"
      >
        <Grid2 container spacing={2}>
          {dropdowns.map((dropdown, index) => (
            <Grid2 size={4} key={index}>
              <MultiSelectDropdown
                selectedOption={dropdown.selectedOption}
                onChange={dropdown.onChange}
                options={dropdown.options}
                label={dropdown.label}
                multiple
                checkBox={true}
                size="small"
                zIndex="99999"
                height="40px"
              />
            </Grid2>
          ))}
        </Grid2>
        <Box
          sx={{
            display: "flex",
            paddingTop: "30px",
            gap: "20px",
            justifyContent: "center",
          }}
        >
          <CommonButton variant="outlined" height="auto" width="auto" onClick={handleCloseModal}>
            Cancel
          </CommonButton>
          <CommonButton disabled={!isDataAvailable} width="auto" onClick={handleDownloadClick}>
            Download
          </CommonButton>
        </Box>
      </CommonModal>
      {/* unwanted */}
      <BasicModal
        open={isRecallModalOpen}
        text={recallModalMessage}
        type={"warning"}
        buttonTextTwo="No"
        handleAddMore={() => setIsRecallModalOpen(false)}
        buttonText={isRecallProcessing ? "Processing..." : "Yes"}
        buttonDisabled={!recallComment || isRecallProcessing}
        buttonTwoDisabled={isRecallProcessing}
        showAdditionalButton={true}
      >
        <TextareaAutosize
          aria-label="minimum height"
          minRows={3}
          placeholder="Comments"
          style={styles.Textarea}
          value={recallComment}
          onChange={(e) => setRecallComment(e.target.value)}
        />
      </BasicModal>
      {/* delete rate modal */}
      <BasicModal
        open={deleteAll.modal}
        text={`${selectedRows.length > 1 ? "Rates" : "Rate"} deleted sucessfully`}
        handleSuccessModal={() => {
          setDeleteAll((prev) => ({ ...prev, loading: false, modal: false }));
          fetchData();
        }}
        buttonText="Done"
      />
      <BasicModal
        open={isDeletePopupOpen}
        text={sourceRateMessage.deleteAlertMsg}
        type="warning"
        handleSuccessModal={onClickDeleteAllHandler}
        handleAddMore={handleCancelDelete}
        buttonText={isProcessing ? "Processing..." : "Yes"}
        buttonTextTwo="No"
        showAdditionalButton={true}
        buttonDisabled={deleteAll.loading}
        buttonTwoDisabled={deleteAll.loading}
      />
    </>
  );
};

export default TimeBaseOfferRateTable;
const styles = {
  Textarea: {
    border: `1px solid ${Colors.tpShadeGray}`,
    borderRadius: "4px",
    fontFamily: "Space Grotesk",
    fontWeight: "600",
    color: Colors.tableBorderColor,
    fontSize: "14px",
    paddingTop: "20px",
    paddingLeft: "14px",
    backgroundColor: Colors.light,
    width: "100%",
    outline: "none",
    resize: "none",
  },
  fadeInAnimation: {
    "@keyframes fadeIn": {
      "0%": {
        opacity: 0,
        transform: "translateY(-10px)",
      },
      "100%": {
        opacity: 1,
        transform: "translateY(0)",
      },
    },
    animation: "fadeIn 0.5s ease-in-out",
    fontSize: "14px",
    whiteSpace: "pre",
    fontWeight: 600,
    padding: "15px 15px",
  },
  deleteAllIcon: {
    cursor: "pointer",
    margin: "0 8px 4px 8px",
    width: "12px",
    height: "18px",
  },
};
const commonMultiSelectProps = {
  multiple: true,
  checkBox: true,
  showSelectAllOption: true,
  chipWidth: "75px",
  zIndex: "99999",
  backgroundColor: `transparent`,
  size: "small",
  showTooltip: true,
  allOptionLabel: "Select All",
};

/* eslint-disable react-hooks/exhaustive-deps */
import { useCallback, useEffect, useMemo, useRef } from "react";
import { useNavigate, useSearchParams } from "react-router-dom";
import { useContext, useState } from "react";
import { DataContext } from "../../../../context/DataContext";
import Item from "@mui/material/Stack";
import { Box, IconButton, InputAdornment, Stack, Grid2, Popover, Tooltip } from "@mui/material";
import { Colors } from "@/theme/colors";
import DataGridCommponent from "@/common/DataGridComponent/DataGridCommponent";
import CustomPaginationComponent from "@/common/Pagination/CustomPaginationComponent";
import TextFieldComponent from "@/common/TextField/TextFieldComponent";
import CommonButton from "@/common/Button/CommonButton";
import ResultPerPageComponent from "@/common/Pagination/ResultPerPageComponent";
import BasicModal from "@/common/Modal/Modal";
import MultiSelectDropdown from "@/common/MultiselectDropdown.js/MultiSelect";
import { SearchIcon } from "../../../../assets/Icons";
import { DownloadIcon } from "../../../../assets/Icons";
import {
  checkMenuEnabled,
  displayToast,
  downloadData,
  filterMVByType,
  findMenuItemByDescription,
  formatNumberWithCommasAndDecimals,
  getLedgerStatusColors,
  updateMetadata,
  useMasterValuesByType,
  useSideMenu,
} from "@/utils/helperFunctions";
import {
  active,
  ERROR_MESSAGES,
  INACTIVE,
  INITIATED,
  LIQUIDITY,
  partnerTypeMap,
  partnerTypeOptions,
  PENDING_APPROVAL,
  suspend,
  terrapayFileName,
  terrpay,
  TPMU,
} from "@/utils/ConstantLabel/Constant";

import LiquidityAPIs from "@/services-(general)/Treasury-Api/LiquidityApiCall";
import IOSSwitch from "@/common/Switch/CustomSwitch";
import { alphanumericWithSpaceHyphen } from "../../../../utils/CommonValidation";
import { useDispatch, useSelector } from "react-redux";
import { setPrefundType, setStatus } from "@/redux/commonSlice";
import useTreasury from "../../../../hooks/useTreasury";
import { MENU_TARGETS, menuIDMap, REQUIRED_PERMISSIONS } from "../../../../utils/constants";
import { Download as WhiteDownloadIcon } from "@mui/icons-material";
import CommonModal from "@/common/Modal/CommonModal";
import { paths } from "@/routes/paths";
import useDebouncer from "@/hooks/useDebouncer";
import InfoIcon from "@mui/icons-material/Info";
import Dropdown from "@/common/Dropdown/Dropdown";
import FeedbackModal from "@/common/Modal/FeedbackModal";
import { setSelectedEntity } from "@redux/entitySlice";

function CommonLedgerTable() {
  const {
    // deleteLedger,
    getEntityList,
    getResidityEntityList,
    ledgerGrid,
    ledgerStatusUpdate,
    terrapaygridDownload,
    getEntityBasedPartnerData,
  } = LiquidityAPIs();
  const navigate = useNavigate();
  const dispatch = useDispatch();
  const selectedEntity = useSelector((state) => state.entity?.selectedEntity);
  const [data, setData] = useState([]);
  const [page, setPage] = useState(1);
  const [limit, setLimit] = useState(10);
  const [totalRowsCount, setTotalRowsCount] = useState();
  const [isLoading, setIsLoading] = useState(false);
  const [entityName, setEntityName] = useState([]);
  const [statusToggle, setStatusToggle] = useState(false);
  const [ledgerName, setLedgerName] = useState(null);
  const [deleteModal, setDeleteModal] = useState(false);
  const [sortInitialState, setSortInitialState] = useState([]);
  const [orderBy, setOrderBy] = useState("");
  const [sortBy, setSortBy] = useState("");
  const [ledgerBalanceSort, setLedgerBalanceSort] = useState(null);
  const [isProcessing, setIsProcessing] = useState(false);
  const [isUpdatePopupOpen, setUpdatePopupOpen] = useState(false);
  const [successMessage, setSuccessMessage] = useState("");
  const {
    resultPerPage,
    selectedCurrencyOptions,
    setSelectedCurrencyOptions,
    selectedLegerTypeOptions,
    setSelectedLedgerTypeOptions,
    selectedEntityNameOptions,
    setSelectedEntityNameOptions,
    selectedPrefundTypeOptions,
    setSelectedPrefundTypeOptions,
    selectedStatusOptions,
    setSelectedStatusOptions,
    terrapayLedgerSearch,
    setTerrpayLedgerSearch,
    setLedgerDetails,
    isStatusUserChanged,
    setIsStatusUserChanged,
    recvPartnerNameOptions,
    setRecvPartnerNameOptions,
  } = useContext(DataContext);
  const [isOpen, setIsOpen] = useState(false);
  const [isDataAvailable, setIsDataAvailable] = useState(false);
  const [anchorE2, setAnchorE2] = useState(null);
  const [searchParams, setSearchParams] = useSearchParams();
  const [selectedRows, setSelectedRows] = useState([]);
  const [updateAll, setUpdateAll] = useState({
    loading: false,
    modal: false,
  });
  const [pendingUpdateInfo, setPendingUpdateInfo] = useState(null);
  const [selectedPartnerTypeOptions, setSelectedPartnerTypeOptions] = useState([]);
  const didAutoSelect = useRef(false);

  const { permissions } = useTreasury(menuIDMap?.TREASURY_LADGER_MANAGEMENT);

  const prefundTypes = useSelector((state) => filterMVByType(state, "LEDGER_PREFUND_TYPES"));

  const ledgerStatusRaw = useMasterValuesByType("LEDGER_STATUS");

  const prefundType = useSelector((state) => state.commonReducer.prefundType);
  // const status = useSelector((state) => state.commonReducer.status);
  const currencyList = useSelector((state) => state.commonReducer.currencyList);
  const sideMenu = useSideMenu();

  const [status] = useState(() => ledgerStatusRaw
    ? [...ledgerStatusRaw]
        .sort((a, b) => a.master_value_display_order - b.master_value_display_order)
        .map((type) => ({
          value: type.master_value_value,
          label: type.master_value_name,
        }))
    : []);
  const [statusReady, setStatusReady] = useState(false);

  // ----- DEFAULT STATUS SETUP -----
  const defaultStatusSet = useRef(false);
  useEffect(() => {
    if (!status?.length) return;

    // If user already changed it, we're good
    if (isStatusUserChanged?.terrapay) {
      setStatusReady(true);
      return;
    }

    // Apply defaults once
    if (
      !defaultStatusSet.current &&
      (!selectedStatusOptions || selectedStatusOptions.length === 0)
    ) {
      const activeStatus = status.find((s) => s.value === active);
      const suspendStatus = status.find((s) => s.value === suspend);
      const defaults = [activeStatus, suspendStatus].filter(Boolean);
      setSelectedStatusOptions(defaults);
      defaultStatusSet.current = true;
    }

    setStatusReady(true);
  }, [status, selectedStatusOptions, isStatusUserChanged]);

  const treasuryDashboard = findMenuItemByDescription(sideMenu, MENU_TARGETS.treasury);
  const permissionsAccess = checkMenuEnabled(
    treasuryDashboard?.children || [],
    MENU_TARGETS.ledgerManagement,
    REQUIRED_PERMISSIONS
  );

  useEffect(() => {
    updateMetadata(prefundTypes, prefundType, setPrefundType, dispatch);
  }, [prefundTypes, dispatch, prefundType]);

  const ledgerBookTypeRaw = useMasterValuesByType("LEDGER_BOOK_TYPE");
  const partnerTypeRaw = useMasterValuesByType("LEDGER_LISTING_CONFIG");

  // Helper function to filter status options based on row data
  const getFilteredStatusOptions = (rowStatus, ledgerBalance) => {
    const numericBalance = Number(ledgerBalance);

    if (rowStatus === active) {
      return numericBalance === 0
        ? status.filter((option) =>
            [active, LIQUIDITY.DELETED, suspend, INACTIVE].includes(option.value)
          )
        : status.filter((option) => [active, suspend, INACTIVE].includes(option.value));
    } else if (rowStatus === suspend) {
      return status.filter((option) => [active, suspend].includes(option.value));
    } else if (rowStatus === INACTIVE) {
      return status.filter((option) => [active, INACTIVE].includes(option.value));
    } else if (rowStatus === INITIATED) {
      return numericBalance === 0
        ? status.filter((option) => [active, INITIATED, LIQUIDITY.DELETED].includes(option.value))
        : status.filter((option) => [active, INITIATED].includes(option.value));
    }
    return status;
  };
  const ledgerBookTypeOptions = useMemo(() => ledgerBookTypeRaw
    ? [...ledgerBookTypeRaw]
        .sort((a, b) => a.master_value_display_order - b.master_value_display_order)
        .map((type) => ({
          value: type.master_value_value,
          label: type.master_value_name,
        }))
    : []);

  const ledgerPartnerTypeOptions = partnerTypeRaw
    ? [...partnerTypeRaw]
        .sort((a, b) => a.master_value_display_order - b.master_value_display_order)
        .map((type) => ({
          value: type.master_value_value,
          label: type.master_value_name,
        }))
    : [];

  const currency = currencyList
    ? [...currencyList]
        .sort((a, b) =>
          a.currency_abbreviation
            .replace(/\s/g, "")
            .localeCompare(b.currency_abbreviation.replace(/\s/g, ""), "en", {
              sensitivity: "base",
            })
        )
        .map((currency) => ({
          value: currency.currency_code,
          label: `${currency.currency_abbreviation} (${currency.currency_code})`,
        }))
    : [];

  const handleLimitChange = useCallback((e) => {
    setLimit(e.target.value);
  }, []);

  const handlePageChange = useCallback((newPage) => {
    setPage(newPage);
  }, []);

  const debouncedSearchTerm = useDebouncer(terrapayLedgerSearch, 800);

  // Auto-select partner types once options are loaded
  useEffect(() => {
    if (
      !didAutoSelect.current &&
      ledgerPartnerTypeOptions.length > 0 &&
      (!selectedPartnerTypeOptions || selectedPartnerTypeOptions.length === 0)
    ) {
      setSelectedPartnerTypeOptions(ledgerPartnerTypeOptions);
      didAutoSelect.current = true;
    }
  }, [ledgerPartnerTypeOptions, selectedPartnerTypeOptions]);

  const buildRequestData = () => {
    const requestData = { ...(debouncedSearchTerm && { search: debouncedSearchTerm }) };

    if (Array.isArray(selectedPartnerTypeOptions) && selectedPartnerTypeOptions.length) {
      requestData.ledgerConfig = selectedPartnerTypeOptions.map((o) => o.value);
    }

    const currentEntity = selectedEntity?.value || selectedEntityNameOptions?.value || null;
    const isValidEntity = entityName.some(
      (e) => e.value?.toUpperCase() === currentEntity?.toUpperCase()
    );
    if (currentEntity && isValidEntity) requestData.ledgerEntityName = [currentEntity];

    if (Array.isArray(selectedLegerTypeOptions) && selectedLegerTypeOptions.length) {
      requestData.ledgerBookType = selectedLegerTypeOptions.map((o) => o.value);
    }

    if (Array.isArray(recvPartnerNameOptions) && recvPartnerNameOptions.length) {
      requestData.partnerId = recvPartnerNameOptions.map((o) => o.value);
    }

    if (Array.isArray(selectedCurrencyOptions) && selectedCurrencyOptions.length) {
      requestData.ledgerCurrency = selectedCurrencyOptions.map((o) => o.value);
    }

    if (Array.isArray(selectedPrefundTypeOptions) && selectedPrefundTypeOptions.length) {
      requestData.prefundType = selectedPrefundTypeOptions.map((o) => o.value);
    }

    if (Array.isArray(selectedStatusOptions) && selectedStatusOptions.length) {
      requestData.ledgerStatus = selectedStatusOptions.map((o) => o.value);
    }

    if (ledgerBalanceSort) {
      requestData.ledgerBalanceSort = ledgerBalanceSort;
    } else if (sortBy && orderBy) {
      requestData.orderBy = orderBy;
      requestData.sortBy = sortBy;
    }

    return requestData;
  };

  const fetchLedgerData = () => {
    const requestData = buildRequestData();
    ledgerGrid(requestData, page, limit)
      .then((res) => {
        const modifiedData = res?.data?.data?.data.map((item, index) => ({
          ...item,
          slno: (page - 1) * limit + index + 1,
        }));
        setData(modifiedData);
        setTotalRowsCount(res?.data?.data?.count);
        setIsDataAvailable(res?.data?.data?.count > 0);
        setIsLoading(false);
      })
      .catch((error) => {
        console.error(error);
        setIsDataAvailable(false);
        setData([]);
        displayToast(error?.response?.data?.message);
      })
      .finally(() => {
        setIsLoading(false);
      });
  };

  useEffect(() => {
    // if (!isEntityReady) return;
    if (!statusReady) return;
    setIsLoading(true);
    fetchLedgerData();
  }, [
    // isEntityReady,
    statusReady,
    debouncedSearchTerm,
    page,
    limit,
    selectedCurrencyOptions,
    selectedEntityNameOptions,
    selectedLegerTypeOptions,
    selectedPrefundTypeOptions,
    selectedStatusOptions,
    selectedPartnerTypeOptions,
    orderBy,
    sortBy,
    ledgerBalanceSort,
    recvPartnerNameOptions,
  ]);

  useEffect(() => {
    setPage(1);
  }, [
    debouncedSearchTerm,
    limit,
    selectedCurrencyOptions,
    selectedEntityNameOptions,
    selectedLegerTypeOptions,
    selectedPrefundTypeOptions,
    selectedStatusOptions,
  ]);

  const handleEnitityName = (event, value) => {
    setSelectedEntityNameOptions(value);
    dispatch(setSelectedEntity(value));
  };

  // Update selected status options based on user input.
  const handleStatus = (event, value) => {
    // setIsStatusUserChanged(true);
    setIsStatusUserChanged((prev) => ({
      ...prev,
      terrapay: true,
    }));
    setSelectedStatusOptions(Array.isArray(value) ? value : null);
  };
  const handleSearchChange = (event) => {
    const inputValue = event.target.value.trimStart();

    if (alphanumericWithSpaceHyphen(inputValue)) {
      setTerrpayLedgerSearch(inputValue);
      searchParams.set("page", 1);
      setSearchParams(searchParams, { replace: true });
    } else {
      displayToast(ERROR_MESSAGES.SEARCH_FIELD_MESSAGE, "warning");
    }
  };

  // useEffect(() => {
  //   // getEntityList()
  //   getResidityEntityList()
  //     .then((res) => {
  //       if (res?.data?.isSuccess) {
  //         const data = res?.data?.data;
  //         const sortedData = data.sort((a, b) => {
  //           const nameA =
  //             a.organization_full_name && a.organization_full_name !== "null"
  //               ? a.organization_full_name.replace(/\s/g, "")
  //               : "";
  //           const nameB =
  //             b.organization_full_name && b.organization_full_name !== "null"
  //               ? b.organization_full_name.replace(/\s/g, "")
  //               : "";
  //           return nameA.localeCompare(nameB, "en", { sensitivity: "base" });
  //         });
  //         const options = sortedData.map((option) => {
  //           const orgName =
  //             option.organization_full_name &&
  //             option.organization_full_name !== "null"
  //               ? option.organization_full_name
  //               : "";
  //           return {
  //             value: option.name,
  //             label: ` ${orgName} (${option.name})`,
  //             cCode: option.country_code,
  //           };
  //         });
  //         setEntityName(options);

  //         const tpEntity = options.find((opt) => opt.value === "TPMU");
  //         if (tpEntity) {
  //           setSelectedEntityNameOptions([tpEntity]);
  //           dispatch(
  //             setSelectedEntity({
  //               value: tpEntity.value,
  //               label: tpEntity.label,
  //             })
  //           );
  //         }

  //         setIsEntityReady(true);
  //       }
  //     })
  //     .catch((err) => {
  //       displayToast(err?.response?.data?.message);
  //       setIsEntityReady(true);
  //     });
  // }, []);

  const handleSuccessModal = () => {
    setStatusToggle(false);
    setSuccessMessage("");
    fetchLedgerData();
  };
  const handleSuccessDeleteModal = () => {
    setDeleteModal(false);
    fetchLedgerData();
  };
  // Function to handle the click event for downloading Terrapayledger data as a CSV file
  const handleDownloadClick = () => {
    const requestData = {
      // search: terrapayLedgerSearch,
      ...(Array.isArray(selectedPartnerTypeOptions) && selectedPartnerTypeOptions.length > 0
        ? { ledgerConfig: selectedPartnerTypeOptions.map((opt) => opt.value) }
        : {}),
    };

    const currentEntity = selectedEntity?.value || selectedEntityNameOptions?.value || null;

    const isValidEntity = entityName.some(
      (entity) => entity.value?.toUpperCase() === currentEntity?.toUpperCase()
    );

    if (currentEntity && isValidEntity) {
      requestData.ledgerEntityName = [currentEntity];
    }
    if (terrapayLedgerSearch) {
      requestData.search = terrapayLedgerSearch;
    }
    if (Array.isArray(selectedLegerTypeOptions) && selectedLegerTypeOptions.length > 0) {
      const selectedLedgerType = selectedLegerTypeOptions.map((option) => option.value);
      requestData.ledgerBookType = selectedLedgerType;
    }
    if (Array.isArray(selectedCurrencyOptions) && selectedCurrencyOptions.length > 0) {
      const selectedCurrency = selectedCurrencyOptions.map((option) => option.value);
      requestData.ledgerCurrency = selectedCurrency;
    }
    if (Array.isArray(selectedPrefundTypeOptions) && selectedPrefundTypeOptions.length > 0) {
      const selectedPrefundType = selectedPrefundTypeOptions.map((option) => option.value);
      requestData.prefundType = selectedPrefundType;
    }
    if (Array.isArray(selectedStatusOptions) && selectedStatusOptions.length > 0) {
      const selectedStatus = selectedStatusOptions.map((option) => option.value);
      requestData.ledgerStatus = selectedStatus;
    }
    if (Array.isArray(recvPartnerNameOptions) && recvPartnerNameOptions.length > 0) {
      requestData.partnerId = recvPartnerNameOptions.map((opt) => opt.value);
    }

    terrapaygridDownload(requestData, orderBy, sortBy)
      .then((response) => {
        downloadData(response, terrapayFileName);
        setIsOpen(false);
      })
      .catch((error) => {
        displayToast(error?.response?.data?.message);
        setIsOpen(false);
      });
  };

  const handleCloseModal = () => {
    setIsOpen(false);
  };

  const handleLedgerDetails = (rowData) => {
    setLedgerDetails(rowData);
    navigate(`${paths?.forex?.main}/${paths?.forex?.viewLedgerTabDetails}`);
  };

  const handleRowSelection = (selectionModel) => {
    setSelectedRows(selectionModel);
  };
  const handleStatusChanges = (e, selectedStatus, rowId) => {
    const newStatus = selectedStatus?.value;
    const ledgerIds = selectedRows && selectedRows.length > 0 ? selectedRows : [rowId];

    if (newStatus === LIQUIDITY.DELETED) {
      for (let id of ledgerIds) {
        const row = data.find((r) => r.ledger_book_id === id);
        const numericBalance = Number(row?.ledgerBalance?.balance) || 0;
        if (numericBalance !== 0) {
          displayToast(
            `Ledger ID ${id} cannot be deleted because its balance is not 0.`,
            "warning"
          );
          return;
        }
      }
    }
    setPendingUpdateInfo({ newStatus, ledgerIds });
    setUpdatePopupOpen(true);
  };

  const onClickUpdateAllHandler = () => {
    if (!pendingUpdateInfo) return;
    setIsProcessing(true);
    const payload = {
      ledgerBookIdList: pendingUpdateInfo.ledgerIds,
      status: pendingUpdateInfo.newStatus,
    };

    setUpdateAll({ ...updateAll, loading: true });

    ledgerStatusUpdate(payload)
      .then(() => {
        const specialTypes = [LIQUIDITY.SBANK_FullName, LIQUIDITY.CBANK_FullName];
        const firstRow = data.find((r) => r.ledger_book_id === pendingUpdateInfo.ledgerIds[0]);
        const message =
          firstRow && specialTypes.includes(firstRow.ledger_book_type)
            ? LIQUIDITY?.statusSentAppMsg
            : LIQUIDITY?.statusUpdateMsg;
        setSuccessMessage(message);
        setStatusToggle(true);
      })
      .catch((error) => {
        setIsProcessing(false);
        displayToast(error?.response?.data?.message);
      })
      .finally(() => {
        setUpdateAll((prev) => ({ ...prev, loading: false }));
        setIsProcessing(false);
        setUpdatePopupOpen(false);
        setPendingUpdateInfo(null);
      });
  };

  const renderStatusCell = ({
    row,
    statusOptions,
    permissions,
    permissionsAccess,
    handleStatusChanges,
  }) => {
    const rowStatus = row.status;
    const rowId = row.ledger_book_id;
    const ledgerBalance = row.ledgerBalance?.balance || 0;
    const editAllowed = row.editAllowed;

    const filteredOptions = getFilteredStatusOptions(rowStatus, ledgerBalance).map((option) => {
      let color;
      switch (option.value?.toUpperCase()) {
        case "ACTIVE":
          color = Colors.tpGreen;
          break;
        case "INACTIVE":
          color = Colors.tpInactive;
          break;
        case "DELETED":
          color = Colors.tpRejectColor;
          break;
        case "SUSPEND":
          color = Colors.tpSuspend;
          break;
        default:
          color = Colors.tpBlue;
      }
      return { ...option, color };
    });

    if (rowStatus === PENDING_APPROVAL) {
      return <div style={{ color: Colors.tpYellow }}>{rowStatus}</div>;
    }

    if (rowStatus === LIQUIDITY.DELETED) {
      return <div style={{ color: Colors.tpRejectColor }}>{rowStatus}</div>;
    }

    const selectedOption = statusOptions.find((s) => s.value === rowStatus);
    const canEdit = permissionsAccess.EDIT;
    const disabled = !editAllowed || !canEdit;

    const toastMsg = "You do not have sufficient privilege. Please contact admin.";
    const { borderColor, labelColor } = getLedgerStatusColors(rowStatus);
    const selectedWithColor = {
      ...selectedOption,
      labelColor,
    };

    return (
      <Box sx={{ position: "relative", display: "inline-block", width: 140 }}>
        <Dropdown
          size="small"
          height="30px"
          width="140px"
          options={filteredOptions}
          label="Status"
          selectedOption={selectedWithColor}
          value={selectedWithColor}
          onChange={(e, value) => {
            if (!disabled) {
              handleStatusChanges(e, value, rowId);
            }
          }}
          disabled={disabled}
          borderColor={borderColor}
          showClearIcon={false}
        />
        {!canEdit && (
          <Box
            onClick={(e) => {
              e.stopPropagation();
              displayToast(toastMsg);
            }}
            sx={{
              position: "absolute",
              top: 0,
              left: 0,
              right: 0,
              bottom: 0,
              cursor: "not-allowed",
              zIndex: 1,
            }}
          />
        )}
      </Box>
    );
  };

  // Terrapay ledger column array
  const cols = [
    {
      field: "slno",
      headerName: "SL No.",
      minWidth: 80,
      maxWidth: 110,
      sortable: false,
      flex: 2,
      renderCell: (cellValues) => {
        return <div>{cellValues?.value ? cellValues?.value : "-"}</div>;
      },
    },
    {
      field: "ledger_book_name",
      headerName: "Ledger Name",
      minWidth: 170,
      sortable: true,
      flex: 1,
      renderCell: (cellValues) => {
        return (
          <div>
            <span>{cellValues?.value ? cellValues?.value : "-"}</span>
          </div>
        );
      },
    },

    {
      field: "ledger_book_id",
      headerName: "Ledger Book ID",
      minWidth: 200,
      sortable: true,
      flex: 1,
      renderCell: (cellValues) => {
        const ledgerBookId = cellValues.value;
        return (
          <div>
            <span
              style={{ textDecoration: "underline", cursor: "pointer" }}
              onClick={() => handleLedgerDetails(cellValues.row)}
            >
              {ledgerBookId ? ledgerBookId : "-"}
            </span>
          </div>
        );
      },
    },
    {
      field: "ledger_book_type",
      headerName: (
        <Box mt={1}>
          <MultiSelectDropdown
            label={"Ledger Type"}
            width={"170px"}
            selectedOption={selectedLegerTypeOptions}
            options={ledgerBookTypeOptions}
            onChange={(e, val) => {
              setSelectedLedgerTypeOptions(val);
              // setPage(1);
              // searchParams.set("page", 1);
              // setSearchParams(searchParams, { replace: true });
            }}
            checkBox
            multiple
            zIndex={1300}
            chipWidth="80px"
            size="small"
          />
        </Box>
      ),
      minWidth: 240,
      sortable: false,
      flex: 1,
      renderCell: (cellValues) => {
        return (
          <div>
            <span>{cellValues?.value ? cellValues?.value : "-"}</span>
          </div>
        );
      },
    },
    {
      field: "organization_full_name",
      headerName: "Entity Name",
      minWidth: 230,
      sortable: false,
      flex: 1,
      renderCell: (cellValues) => {
        const partnername = cellValues.row.entityName?.organization_full_name || "-";
        return (
          <div>
            <span>{partnername}</span>
          </div>
        );
      },
    },
    {
      field: "partner_name",
      headerName: "Partner Name",
      minWidth: 170,
      sortable: false,
      flex: 1,
      renderCell: (cellValues) => {
        return <div style={{ wordBreak: "break-word" }}>{cellValues.value || "-"}</div>;
      },
    },
    {
      field: "currency",
      headerName: (
        <Box mt={1}>
          <MultiSelectDropdown
            label={"Currency"}
            width={"170px"}
            selectedOption={selectedCurrencyOptions}
            options={currency}
            onChange={(e, val) => {
              setSelectedCurrencyOptions(val);
              // setPage(1);
              // searchParams.set("page", 1);
              // setSearchParams(searchParams, { replace: true });
            }}
            checkBox
            multiple
            zIndex={1300}
            chipWidth="80px"
            size="small"
          />
        </Box>
      ),
      minWidth: 190,
      flex: 1,
      sortable: false,
      renderCell: (cellValues) => {
        return (
          <div>
            <span> {cellValues?.value ? cellValues?.value : "-"}</span>
          </div>
        );
      },
    },
    {
      field: "Balance",
      headerName: "Balance",
      minWidth: 180,
      maxWidth: 200,
      headerAlign: "center",
      align: "right",
      flex: 1,
      sortable: true,
      renderCell: (cellValues) => {
        const balance = cellValues.row.ledgerBalance?.balance || "0";
        const formattedBalance = formatNumberWithCommasAndDecimals(balance);
        const balanceStyle = balance !== "-" ? { textAlign: "right" } : { paddingRight: "60px" };
        return (
          <div>
            <span style={balanceStyle}>{formattedBalance}</span>
          </div>
        );
      },
    },
    {
      field: "status",
      headerName: "Status",
      minWidth: 170,
      maxWidth: 180,
      flex: 2,
      sortable: false,
      renderCell: (cellValues) =>
        renderStatusCell({
          row: cellValues.row,
          statusOptions: status,
          permissions,
          permissionsAccess,
          handleStatusChanges,
        }),
    },
  ];

  if (permissions.hasEditAccess) {
    cols.push({
      field: "edit",
      headerName: "",
      width: 50,
      headerAlign: "center",
      align: "center",
      sortable: false,
      renderCell: (cellValues) => {
        const ledgerBookId = cellValues.row.ledger_book_id;
        const ledgerBookType = cellValues.row.ledger_book_type;
        const isDeleted = cellValues.row.status === "DELETED";
        const isPendingApproval = cellValues.row.status === "PENDING_APPROVAL";
        const editAllowed = cellValues.row.editAllowed;
        if (isDeleted || isPendingApproval || !editAllowed) {
          return <div style={{ color: "grey", cursor: "not-allowed" }}>Edit</div>;
        }
        const handleClick = () => {
          navigate(`/forex/editledger/${ledgerBookId}`, {
            state: { ledgerBookId, ledgerBookType, edit: true },
          });
        };

        return (
          <div style={{ cursor: "pointer", textDecoration: "underline" }} onClick={handleClick}>
            Edit
          </div>
        );
      },
    });
  }
  cols.push({
    field: "view",
    headerName: "",
    width: 50,
    headerAlign: "center",
    align: "center",
    sortable: false,
    renderCell: (cellValues) => {
      const ledgerBookId = cellValues.row.ledger_book_id;
      const ledgerBookType = cellValues.row.ledger_book_type;

      const handleClick = () => {
        navigate(`/forex/editledger/${ledgerBookId}`, {
          state: { ledgerBookId, ledgerBookType },
        });
      };

      return (
        <div style={{ cursor: "pointer", textDecoration: "underline" }} onClick={handleClick}>
          <Tooltip title="View Ledger" placement="top">
            <InfoIcon sx={{ color: Colors?.tpBlue }} />
          </Tooltip>
        </div>
      );
    },
  });

  const handleClose2 = () => {
    setAnchorE2(null);
  };

  const open = Boolean(anchorE2);
  const id = open ? "simple-popover" : undefined;
  const [isPartnerDropdownDisabled, setIsPartnerDropdownDisabled] = useState(false);

  const [partnerListOpt, setPartnerListOpt] = useState([]);

  let DEFAULT_ENTITY = selectedEntity ? selectedEntity.value || selectedEntity : TPMU;

  useEffect(() => {
    const selectedTypes =
      selectedPartnerTypeOptions?.map((t) => t.label?.toLowerCase()?.trim()) || [];

    const isOnlyTerraPay = selectedTypes.length === 1 && selectedTypes[0] === "terrapay";
    const isEmpty = selectedTypes.length === 0;

    const shouldDisablePartnerDropdown = isOnlyTerraPay || isEmpty;
    setIsPartnerDropdownDisabled(shouldDisablePartnerDropdown);

    if (shouldDisablePartnerDropdown || !DEFAULT_ENTITY) {
      setPartnerListOpt([]);
      setRecvPartnerNameOptions(null);
      return;
    }

    const mappedTypes = ledgerPartnerTypeOptions
      .filter(
        (opt) =>
          selectedTypes.includes(opt.label.toLowerCase()) && opt.label.toLowerCase() !== "terrapay"
      )
      .map((opt) => opt.value);

    const commaSeparatedType = mappedTypes.join(",");

    if (commaSeparatedType) {
      getEntityBasedPartnerData(DEFAULT_ENTITY, commaSeparatedType)
        .then((res) => {
          const formattedList =
            res.data?.data?.map((partner) => ({
              label: partner.partner_name.trim(),
              value: partner.partner_id,
            })) || [];

          const sortedList = formattedList.sort((a, b) =>
            a.label.localeCompare(b.label, undefined, { sensitivity: "base" })
          );

          if (JSON.stringify(sortedList) !== JSON.stringify(partnerListOpt)) {
            setPartnerListOpt(sortedList);
          }
        })
        .catch((err) => {
          displayToast(err?.response?.data?.message);
        });
    }
  }, [selectedPartnerTypeOptions, DEFAULT_ENTITY]);

  const handlePartnerName = (event, value) => {
    setRecvPartnerNameOptions(Array.isArray(value) ? value : null);
  };

  //   console.log("entityName", entityName);

  const showPartnerTypeDropdown = false;

  const dropdownConfigs = [
    {
      label: "Partner Type",
      options: ledgerPartnerTypeOptions,
      selectedOption: selectedPartnerTypeOptions,
      onChange: (e, val) => setSelectedPartnerTypeOptions(val),
      showSelectAllOption: true,
      hidden: !showPartnerTypeDropdown,
    },

    {
      label: "Partner Name",
      options: partnerListOpt,
      // options: partnerName,
      selectedOption: recvPartnerNameOptions,
      onChange: handlePartnerName,
      disabled: isPartnerDropdownDisabled,
    },
    {
      label: "Status",
      options: status,
      selectedOption: selectedStatusOptions,
      onChange: handleStatus,
    },
  ];

  const handleClickCreateBtn = () => {
    navigate(`${paths?.forex?.main}/${paths?.forex?.createLedger}`, {
      state: "terrapayGrid",
    });
    localStorage.removeItem("TxnType");
  };
  const handleCancelUpdate = () => {
    setUpdatePopupOpen(false);
    setPendingUpdateInfo(null);
  };
  return (
    <>
      <Grid2 container></Grid2>
      <Grid2
        container
        sx={{
          padding: "20px 10px 0px 10px",
        }}
      >
        <Grid2 item size={8.2}>
          <Grid2 container spacing={{ xs: 1, md: 1 }} sx={{ alignItems: "end" }}>
            {dropdownConfigs.map((config, index) => {
              const selectedValueKey = Array.isArray(config.selectedOption)
                ? config.selectedOption.map((o) => o?.value).join(",")
                : config.selectedOption?.value || "none";

              const forceResetKey = `${config.label}-${config.options?.length}-${selectedValueKey}`;

              return (
                <Grid2 item size={2.4} key={forceResetKey}>
                  <Item>
                    {config.label === "Entity Name" ? (
                      <Dropdown
                        key={forceResetKey}
                        size="small"
                        height="38px"
                        width="100%"
                        options={config.options}
                        selectedOption={config.selectedOption}
                        onChange={(e, value) => config.onChange(e, value)}
                        label={config.label}
                        showTooltip
                      />
                    ) : (
                      <MultiSelectDropdown
                        key={index}
                        height="38px"
                        options={config.options}
                        selectedOption={config.selectedOption}
                        onChange={config.onChange}
                        label={config.label}
                        multiple
                        checkBox={true}
                        zIndex="1100"
                        size="small"
                        chipWidth="80px"
                        showSelectAllOption={config.showSelectAllOption}
                        disabled={config.disabled}
                      />
                    )}
                  </Item>
                </Grid2>
              );
            })}

            <Grid2 item size={2.4}>
              <TextFieldComponent
                style={{ minWidth: "180px" }}
                type="text"
                label={"Ledger Book (ID / Name)"}
                value={terrapayLedgerSearch}
                onChange={handleSearchChange}
                InputProps={{
                  endAdornment: (
                    <InputAdornment position="end">
                      <IconButton>
                        <SearchIcon height={"16px"} width={"16px"} />
                      </IconButton>
                    </InputAdornment>
                  ),
                }}
              />
            </Grid2>
          </Grid2>
        </Grid2>

        <Grid2 item size={3.8}>
          <Item>
            <Box
              sx={{
                display: "flex",
                justifyContent: "flex-end",
                columnGap: "16px",
              }}
            >
              <Popover
                id={id}
                open={open}
                anchorEl={anchorE2}
                onClose={handleClose2}
                anchorOrigin={{
                  vertical: "bottom",
                  horizontal: "left",
                }}
              >
                <Box sx={{ p: 2 }}>
                  <TextFieldComponent
                    style={{ minWidth: "120px" }}
                    type="text"
                    label={"Ledger Book (ID / Name)"}
                    value={terrapayLedgerSearch}
                    onChange={handleSearchChange}
                    InputProps={{
                      endAdornment: (
                        <InputAdornment position="end">
                          <IconButton>
                            <SearchIcon height={"16px"} width={"16px"} />
                          </IconButton>
                        </InputAdornment>
                      ),
                    }}
                  />
                </Box>
              </Popover>

              {permissions.hasCreateAccess && (
                <Box onClick={handleClickCreateBtn}>
                  <CommonButton style={{ width: "max-content" }}>Create Ledger</CommonButton>
                </Box>
              )}
              <ResultPerPageComponent
                countPerPage={resultPerPage}
                limit={limit}
                handleLimitChange={handleLimitChange}
              />
              <DownloadIcon
                onClick={() => {
                  if (isDataAvailable) {
                    setIsOpen(true);
                  }
                }}
                fontSize="large"
                style={{
                  cursor: isDataAvailable ? "Pointer" : "Not-Allowed",
                  marginTop: "3px",
                }}
              />
            </Box>
          </Item>
        </Grid2>
      </Grid2>

      <Box mt={3}>
        <DataGridCommponent
          data={data}
          columns={cols}
          getRowId={(r) => r?.ledger_book_id}
          rowColor={Colors?.light}
          colors={Colors}
          customHeight={"auto"}
          loading={isLoading}
          loadingMinHeight="400px"
          callback={(a, b, c) => {
            // setSortBy(c[0]?.field);
            // setOrderBy(c[0]?.sort?.toUpperCase());
            const sortedField = c[0]?.field;
            const sortOrder = c[0]?.sort?.toUpperCase();
            if (sortedField === "Balance") {
              setLedgerBalanceSort(sortOrder);
              setSortBy("");
              setOrderBy("");
            } else {
              setSortBy(sortedField);
              setOrderBy(sortOrder);
              setLedgerBalanceSort(null);
            }
            setSortInitialState(c);
          }}
          sortInitialState={sortInitialState}
          checkboxSelection
          // isRowSelectable={(params) => params.row.status !== "PENDING_APPROVAL" && params.row.status !== "DELETED"}
          isRowSelectable={(params) => {
            if (
              params.row.status === PENDING_APPROVAL ||
              params.row.status === LIQUIDITY.DELETED ||
              params.row.editAllowed === false
            ) {
              return false;
            }
            if (selectedRows.length === 0) {
              return true;
            }
            const firstSelectedId = selectedRows[0];
            const firstSelectedRow = data.find((row) => row.ledger_book_id === firstSelectedId);
            // return firstSelectedRow && params.row.status === firstSelectedRow.status;
            if (!firstSelectedRow || params.row.status !== firstSelectedRow.status) {
              return false;
            }

            const specialTypes = [LIQUIDITY.SBANK_FullName, LIQUIDITY.CBANK_FullName];

            if (selectedRows.length > 0) {
              const anySpecialSelected = selectedRows.some((id) => {
                const row = data.find((r) => r.ledger_book_id === id);
                return row && specialTypes.includes(row.ledger_book_type);
              });

              if (anySpecialSelected) {
                if (!specialTypes.includes(params.row.ledger_book_type)) {
                  return false;
                }
              } else {
                if (specialTypes.includes(params.row.ledger_book_type)) {
                  return false;
                }
              }
            }
            return true;
          }}
          onRowSelectionModelChange={handleRowSelection}
        />
      </Box>
      {isDataAvailable && (
        <Box>
          <CustomPaginationComponent
            totalItemsCount={totalRowsCount}
            page={page}
            numberOfItemsPerPage={limit}
            onChangePage={handlePageChange}
            colors={{ tpBlue: Colors.tpBlue }}
          />
        </Box>
      )}
      <BasicModal
        open={statusToggle}
        handleSuccessModal={handleSuccessModal}
        text={successMessage}
        buttonText="Done"
      />
      <BasicModal
        open={deleteModal}
        handleSuccessModal={handleSuccessDeleteModal}
        // text={`${selectedLedgerName} deleted successfully`}
        buttonText="Done"
      />
      <CommonModal
        open={isOpen}
        handleClose={handleCloseModal}
        title={"Download"}
        width="500px"
        titleIcon={<WhiteDownloadIcon fontSize="large" color="white" />}
        headerPadding="20px 20px"
        fontSize="12px"
      >
        <Grid2 container spacing={2}>
          {dropdownConfigs
            .filter((config) => !config.hidden)
            .map((config, index) => {
              const selectedValueKey = Array.isArray(config.selectedOption)
                ? config.selectedOption.map((o) => o?.value).join(",")
                : config.selectedOption?.value || "none";

              const forceResetKey = `${config.label}-${config.options?.length}-${selectedValueKey}`;

              return (
                <Grid2 item size={6} key={forceResetKey}>
                  <Item>
                    {config.label === "Entity Name" ? (
                      <Dropdown
                        key={forceResetKey}
                        size="small"
                        height="38px"
                        width="100%"
                        options={config.options}
                        selectedOption={config.selectedOption}
                        onChange={(e, value) => config.onChange(e, value)}
                        label={config.label}
                      />
                    ) : (
                      <MultiSelectDropdown
                        height="38px"
                        key={index}
                        options={config.options}
                        selectedOption={config.selectedOption}
                        onChange={config.onChange}
                        label={config.label}
                        disabled={config.disabled}
                        multiple
                        checkBox={true}
                        zIndex="1100"
                        size="small"
                        chipWidth="80px"
                      />
                    )}
                  </Item>
                </Grid2>
              );
            })}
        </Grid2>
        <Box
          sx={{
            display: "flex",
            paddingTop: "30px",
            gap: "20px",
            justifyContent: "center",
          }}
        >
          <CommonButton variant="outlined" height="auto" width="auto" onClick={handleCloseModal}>
            Cancel
          </CommonButton>
          <CommonButton disabled={!isDataAvailable} width="auto" onClick={handleDownloadClick}>
            Download
          </CommonButton>
        </Box>
      </CommonModal>
      <FeedbackModal
        open={isUpdatePopupOpen}
        message={"Are you sure want to update the status ?"}
        type="warning"
        Closable={false}
        handleClose={handleCancelUpdate}
      >
        <Box style={{ display: "flex", gap: "20px", justifyContent: "center" }}>
          <CommonButton
            width="auto"
            onClick={handleCancelUpdate}
            disabled={updateAll.loading}
            variant="outlined"
            style={styles.Buttonstyles}
          >
            No
          </CommonButton>
          <CommonButton
            width="auto"
            onClick={onClickUpdateAllHandler}
            disabled={updateAll.loading}
            variant="outlined"
            style={styles.Buttonstyles}
          >
            {isProcessing ? "Processing..." : "Yes"}
          </CommonButton>
        </Box>
      </FeedbackModal>
    </>
  );
}

export default CommonLedgerTable;

const styles = {
  Buttonstyles: {
    backgroundColor: Colors.tpLightBlue,
    borderColor: Colors.tpBlue,
    color: Colors.tpBlue,
    fontSize: "14px",
    fontWeight: 500,
  },
};

i have shared two components here timebased rate and common ledger table there is a dropdown that is common here partner dropdown in the timebased rate and other pages the partners dropdown is working as expected but in ledger alone in the partner dropdown when a user select an option the dropdown closes and the user needs to click on the dropdown again and select but whereas the other pages it will stay open the user will br able to select multiple options this issue only persists in ledger section only for partners dropdown 
