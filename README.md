select dest country component
import { memo, useCallback } from “react”;
import { Stack, Box, Chip } from “@mui/material”;
import CustomAutocomplete from “@/common/MultiselectDropdown.js/CustomAutocomplete”;
import { FontSmall } from “@/common/Typography/CustomTypography”;
import { Colors } from “@/theme/colors”;
import theme from “@/theme/theme”;
const DestinationCountrySelection = ({
country,
selectedEntities,
selectAllEntities,
countrySelectAll,
isDisabled,
fieldsDisabled,
entityValue,
inputValue,
hasMatches,
errorText,
handleSelectedCountry,
handleDeletedCountry,
setEntityValue,
setCountrySelectAll,
handleCountrySearchInputChange,
}) => {
return (
<Stack direction={“row”} gap={“25px”} width={“100%”} marginTop={“12px”}>
<Stack>
<FontSmall sx={{ paddingBottom: “15px” }}>Select Destination Countries</FontSmall>
<Stack>
<CustomAutocomplete
list={country}
selectedOptions={selectAllEntities ? country : selectedEntities}
handleSelectedOptionsChange={handleSelectedCountry}
autocompleteValue={entityValue}
setAutocompleteValue={setEntityValue}
style={{ width: 350, height: 180, marginTop: “17px” }}
listboxStyle={{ maxHeight: “200px” }}
ListboxProps={{ overflowY: “hidden” }}
AllOption={“Select All”}
DefaultOption={{ title: “Default” }}
width={“350px”}
setSelectAll={setCountrySelectAll}
selectAll={countrySelectAll}
height={“48px”}
onInputChange={handleCountrySearchInputChange}
isOptionDisabled={isDisabled || fieldsDisabled}
withAllOption={country && country.length > 1}
/>
{inputValue && !hasMatches ? (
<div style={{ color: Colors.tpRed, marginTop: “20px”, fontSize: “12px” }}>
No match found
</div>
) : (
errorText && <div style={styles.errorText}>{errorText}</div>
)}
</Stack>
</Stack>
  <Stack width={"100%"}>
    <FontSmall sx={{ paddingBottom: "15px" }}>
      Destination Countries Selected (
      {selectAllEntities ? country.length - 1 : selectedEntities.length})
    </FontSmall>
    <Box sx={styles.chipContainer}>
      {selectedEntities?.map((option) => (
        <Chip
          sx={{
            margin: "5px",
            color: theme.palette.primary.main,
            fontSize: "12px",
          }}
          key={option.code}
          label={option.title}
          variant="outlined"
          onDelete={() => handleDeletedCountry(option)}
        />
      ))}
    </Box>
  </Stack>
</Stack>
);
};
export default memo(DestinationCountrySelection);
const styles = {
chipContainer: {
width: “350px”,
height: “248px”,
border: 1px solid ${Colors?.tpBorderColor},
borderRadius: “4px”,
background: “white”,
padding: “15px”,
overflow: “auto”,
},
errorText: {
color: Colors.tpRed,
marginTop: “228px”,
fontSize: “12px”,
},
addnew rate page where they have used select dest country comp
import DestinationCountrySelection from “./TimeBaseAddNewUI/DestinationCountrySelection”;
<Grid2 item size={12}>
<Box sx={{ minHeight: “500px” }}>
<DestinationCountrySelection
country={country}
selectedEntities={selectedEntities}
selectAllEntities={selectAllEntities}
countrySelectAll={countrySelectAll}
isDisabled={isDisabled}
fieldsDisabled={fieldsDisabled}
entityValue={entityValue}
setEntityValue={setEntityValue}
inputValue={inputValue}
hasMatches={hasMatches}
errorText={errorMessages.desCountryCode}
handleSelectedCountry={handleSelectedCountry}
handleDeletedCountry={handleDeletedCountry}
setCountrySelectAll={setCountrySelectAll}
handleCountrySearchInputChange={handleCountrySearchInputChange}
/>
        <Stack direction={"row"} gap={"25px"} marginTop={"32px"}>
          <Stack>
            <FontSmall
              sx={{
                paddingBottom: "15px",
              }}
            >
              Select Send Partner
            </FontSmall>
            <Stack>
              <CustomAutocomplete
                list={partner}
                selectedOptions={selectedAllPartners ? partner : selectedPartners}
                handleSelectedOptionsChange={handleSelectedPartners}
                autocompleteValue={partnerValue}
                setAutocompleteValue={setPartnerValue}
                listboxStyle={{ maxHeight: "195px" }}
                ListboxProps={{ overflowY: "hidden" }}
                AllOption={"Select All"}
                DefaultOption={{ title: "Default" }}
                width={"350px"}
                PopperComponent={CustomPopper}
                setSelectAll={setPartnerSelectAll}
                selectAll={partnerSelectAll}
                height={"48px"}
                onInputChange={handlePartnerSearchInputChange}
                isOptionDisabled={isDefaultPartnerSelected || fieldsDisabled}
              />
              {partnerInputValue && !hasMatchesPartner ? (
                <div
                  style={{
                    color: Colors.tpRed,
                    marginTop: "20px",
                    fontSize: "12px",
                  }}
                >
                  No match found
                </div>
              ) : (
                errorMessages.srcPartner &&
                !selectedPartners.length && (
                  <div style={styles.errorText}>{errorMessages.srcPartner}</div>
                )
              )}
            </Stack>
          </Stack>
          <Stack width={"100%"}>
            <FontSmall
              sx={{
                paddingBottom: "15px",
              }}
            >
              Send Partner Selected ({selectedPartners?.length})
            </FontSmall>
            <Box sx={styles.chipContainer}>
              {selectedPartners?.map((option) => (
                <Chip
                  sx={styles.chip}
                  key={option.code}
                  label={option.title}
                  variant="outlined"
                  onDelete={() => handleDeletePartners(option)}
                />
              ))}
            </Box>
          </Stack>
        </Stack>

};

custom auto complete comp

import React, { memo, useCallback, useMemo } from “react”;
import {
Checkbox,
TextField,
Autocomplete,
Popper,
InputAdornment,
IconButton,
Backdrop,
CircularProgress,
} from “@mui/material”;
import CheckBoxOutlineBlankIcon from “@mui/icons-material/CheckBoxOutlineBlank”;
import CheckBoxIcon from “@mui/icons-material/CheckBox”;
import { FixedSizeList } from “react-window”;
import “../../styles/autocomplete.css”;
import theme from “../../theme/theme”;
import { SearchIcon } from “../../assets/Icons”;
import { Colors } from “@/theme/colors”;
const icon = <CheckBoxOutlineBlankIcon fontSize="small" />;
const checkedIcon = <CheckBoxIcon fontSize="small" fill="none" />;
const VirtualizedListboxComponent = React.forwardRef(function VirtualizedListboxComponent(
{ children, style, overflowY = “auto”, …rest },
ref
) {
const itemData = React.Children.toArray(children);
const ITEM_SIZE = 50;
const HEIGHT = Math.min(8, itemData.length) * ITEM_SIZE;
return (
<div
ref={ref}
style={{
…style,
backgroundColor: theme.palette.secondary.main,
borderRadius: “4px”,
boxShadow: “1px 1px 1px rgba(0, 0, 0, 0.2)”,
border: “1px solid #A2A8BD”,
maxHeight: “200px”,
overflowY,
fontSize: “12px”,
position: “fixed”,
width: “100%”,
zIndex: 1200,
height: “auto”,
}}
{…rest}
>
<FixedSizeList
height={HEIGHT}
width="100%"
itemSize={ITEM_SIZE}
itemCount={itemData.length}
overscanCount={5}
itemData={itemData}
>
{({ index, style }) =>
React.cloneElement(itemData[index], {
style: {
…style,
top: style.top,
padding: “0 12px”,
},
})
}
</FixedSizeList>
</div>
);
});
function CustomAutocomplete(props) {
const {
disabled,
height,
width,
setSelectAll,
selectAll,
getOptionSelected: customGetOptionSelected,
isOptionEqualToValue: customIsOptionEqualToValue,
onInputChange,
AllOption,
DefaultOption,
handleSelectedOptionsChange,
setAutocompleteValue,
list,
selectedOptions,
listboxStyle,
ListboxProps = {},
isOptionDisabled,
useCustomPopper = true,
withAllOption = true,
fontSize=“16px”,
} = props;
const { overflowY = “auto” } = ListboxProps;
const [openBackdrop, setOpenBackdrop] = React.useState(false);
const handleSelectAllChange = useCallback(
(event) => {
const newSelectAll = event.target.checked;
setOpenBackdrop(true); // show backdrop
setSelectAll(newSelectAll);
const selectedOptions = newSelectAll ? list : [];
handleSelectedOptionsChange(event, selectedOptions);
setTimeout(() => setOpenBackdrop(false), 300); // hide after short delay
},
[list, handleSelectedOptionsChange]
);
const handleOptionsChange = useCallback(
(event, selectedOptions) => {
setOpenBackdrop(true); // show backdrop
  const hasAllOption = selectedOptions.some((option) => option.title === AllOption);
  const hasDefaultOption =
    DefaultOption && selectedOptions.some((option) => option.title === DefaultOption.title);

  if (hasAllOption) {
    setSelectAll(!selectAll);
    handleSelectedOptionsChange(event, selectAll ? [] : list);
  } else if (hasDefaultOption) {
    const isDefaultOptionSelected = selectedOptions.includes(DefaultOption);
    const newSelectedOptions = isDefaultOptionSelected
      ? [DefaultOption, ...list]
      : selectedOptions.filter((option) => option !== DefaultOption);
    handleSelectedOptionsChange(event, newSelectedOptions);
    setAutocompleteValue(newSelectedOptions);
  } else {
    handleSelectedOptionsChange(event, selectedOptions);
    setAutocompleteValue(selectedOptions);
  }

  if (selectedOptions.length === list.length) {
    setSelectAll(true);
  } else if (!hasAllOption) {
    setSelectAll(false);
  }

  setTimeout(() => setOpenBackdrop(false), 300); // hide after short delay
},
[list, selectAll, DefaultOption, handleSelectedOptionsChange, setAutocompleteValue]

);
const optionsWithAllAndDefault = useMemo(() => {
if (!list?.length) {
return DefaultOption ? [DefaultOption] : [];
}
return [
…(DefaultOption ? [DefaultOption] : []),
…(withAllOption ? [{ title: AllOption, selectAll: true }] : []),
…list,
];
}, [list, DefaultOption, withAllOption, AllOption]);
const handleTextFieldKeyDown = (event) => {
if (event.key === “Backspace” && event.target.value === “”) {
event.stopPropagation();
return;
}
// if (event.key === “Enter”) {
//   event.stopPropagation();
//   return;
// }
};
return (
<>
<Backdrop
sx={{ color: Colors.light, zIndex: (theme) => theme.zIndex.drawer + 1 }}
open={openBackdrop}
>
<CircularProgress color="inherit" />
</Backdrop>
<Autocomplete
multiple
id=“checkboxes-tags-demo”
options={optionsWithAllAndDefault}
disableCloseOnSelect
freeSolo
disabled={disabled}
disableClearable
disablePortal={false}
PopperComponent={
useCustomPopper
? ({ style, …props }) => (
<CustomPopper
{…props}
style={{
…style,
zIndex: listboxStyle?.zIndex ? listboxStyle?.zIndex : “initial”,
}}
/>
)
: undefined
}
getOptionLabel={(option) => option.title}
getOptionSelected={
customGetOptionSelected || ((option, value) => option.title === value.title)
}
isOptionEqualToValue={
customIsOptionEqualToValue || ((option, value) => option.title === value.title)
}
onChange={handleOptionsChange}
onInputChange={onInputChange}
renderOption={(props, option, { selected }) => {
const disabledOption =
typeof isOptionDisabled === “function”
? isOptionDisabled(option)
: isOptionDisabled && option.title !== DefaultOption?.title;
      // Remove onClick if disabled
      if (disabledOption) {
        delete props.onClick;
      }

      return (
        <li
          {...props}
          aria-disabled={disabledOption}
          style={{
            ...props.style,
            color: theme.palette.primary.main,
            minHeight: "30px",
            height: "50px",
            pointerEvents: disabledOption ? "none" : "auto",
            opacity: disabledOption ? 0.5 : 1,
            userSelect: disabledOption ? "none" : "auto",
            cursor: disabledOption ? "not-allowed" : "pointer",
          }}
        >
          {option.selectAll ? (
            <Checkbox
              checked={selectAll}
              disabled={disabledOption}
              icon={<CheckBoxOutlineBlankIcon sx={{ width: 20, height: 20 }} />}
              checkedIcon={<CheckBoxIcon sx={{ width: 20, height: 20 }} />}
              style={{
                color: theme.palette.primary.main,
                transform: "scale(1.0)",
              }}
              onChange={handleSelectAllChange}
              inputProps={{
                "aria-label": AllOption || "All",
              }}
            />
          ) : (
            <Checkbox
              icon={icon}
              disabled={disabledOption}
              checkedIcon={checkedIcon}
              style={{
                color: theme.palette.primary.main,
                opacity: disabledOption ? 0.5 : 1, // also fade the checkbox
              }}
              checked={selected}
            />
          )}
          <span style={{ marginTop: "3px" }}>{option.title || option.value}</span>
        </li>
      );
    }}
    renderInput={(params) => (
      <TextField
        {...params}
        disabled={disabled}
        onKeyDown={handleTextFieldKeyDown}
        sx={{
          "& .MuiInputBase-root": {
            backgroundColor: disabled ? "#F0F0F3" : "#FFFFFF",
            height: height || "38px",
            width: width,
          },
          "& .MuiOutlinedInput-notchedOutline": {
            border: "1.8px solid #909BB8",
          },
          "& .MuiOutlinedInput-root:hover fieldset": {
            borderColor: Colors.tpBlue,
          },
          "& .MuiOutlinedInput-root.Mui-focused:hover fieldset": {
            borderColor: Colors.tpBlue,
          },
          input: {
            "&::placeholder": {
              fontSize: "12px",
              color: "#434B66",
            },
            fontSize: fontSize,
            color: Colors.tpBlue,
            fontWeight: 400,
            fontFamily: "Space Grotesk",
          },
        }}
        size="small"
        autoComplete="off"
        placeholder="Search"
        InputProps={{
          ...params.InputProps,
          endAdornment: (
            <>
              {params.InputProps.endAdornment}
              <InputAdornment position="end">
                <IconButton>
                  <SearchIcon />
                </IconButton>
              </InputAdornment>
            </>
          ),
        }}
      />
    )}
    value={selectedOptions}
    renderTags={() => null}
    open
    ListboxComponent={VirtualizedListboxComponent}
    ListboxProps={{ style: listboxStyle, disabled, overflowY }}
  />
</>

);
}
function CustomPopper({ style, …popperProps }) {
return (
<Popper
{…popperProps}
style={{
…style,
zIndex: style.zIndex,
}}
placement=“bottom-start”
/>
);
}
export default memo(CustomAutocomplete);
in the adde new rate page they have used a select dest component
when this dropdown has 1 or 2 options the keyboard selection is working properly if it has more values when im scrolling down it scrolls for a bit then its stopping or something
but when im scrolling with mouse or touchpad its scrolling well
it is a common component being used at many places what we can do to fix that for keyboard dont generate in artifact
show the changes alone 