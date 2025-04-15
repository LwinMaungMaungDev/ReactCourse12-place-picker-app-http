# React Native course by Maximilian Schwarzmüller

To start the backend server, in the backend folder directory, run => **_node app.js_**
To start the Vite frontend, run => **_npm run dev_**

Nothing fancy. We just use **fetch** to request the endpoint. We use the **useEffect** to avoid infinite loop.

```
export default function AvailablePlaces({ onSelectPlace }) {
  const [availablePlaces, setAvailablePlaces] = useState([]);
  useEffect(() => {
    async function fetchPlaces() {
      const res = await fetch("http://localhost:3000/places");
      const resData = await res.json();
      setAvailablePlaces(resData.places);
    }
    fetchPlaces();
  }, []);
  return (
    <Places
      title="Available Places"
      places={availablePlaces}
      fallbackText="No places available."
      onSelectPlace={onSelectPlace}
    />
  );
}

```

One thing to note, in the backend, we do this =>

```
app.use(express.static('images'));
```

so that we can request the image file like this from the frontend =>

```
<img src={`http://localhost:3000/${place.image.src}`} alt={place.image.alt} />
```

## Handling Loading States

We just create another state for loading state and pass it to the _Places_ component.

```
export default function AvailablePlaces({ onSelectPlace }) {
  const [isFetching, setIsFetching] = useState(false);
  const [availablePlaces, setAvailablePlaces] = useState([]);
  useEffect(() => {
    async function fetchPlaces() {
      setIsFetching(true);
      const res = await fetch("http://localhost:3000/places");
      const resData = await res.json();
      setAvailablePlaces(resData.places);
      setIsFetching(false);
    }
    fetchPlaces();
  }, []);
  return (
    <Places
      title="Available Places"
      places={availablePlaces}
      isLoading={isFetching}
      loadingText="Fetching place data..."
      fallbackText="No places available."
      onSelectPlace={onSelectPlace}
    />
  );
}
```

Then inside the _Places_ component, we show the paragraph if it's loading and hide other components.

```
{isLoading && <p className="fallback-text">{loadingText}</p>}
```

## Handling Http Errors

```
import { useEffect, useState } from "react";
import Places from "./Places.jsx";
import Error from "./Error.jsx";
export default function AvailablePlaces({ onSelectPlace }) {
  const [isFetching, setIsFetching] = useState(false);
  const [availablePlaces, setAvailablePlaces] = useState([]);
  const [error, setError] = useState();
  useEffect(() => {
    async function fetchPlaces() {
      setIsFetching(true);
      try {
        const res = await fetch("http://localhost:3000/places");
        const resData = await res.json();
        if (!res.ok) {
          throw new Error("Failed to fetch places");
        }
        setAvailablePlaces(resData.places);
      } catch (err) {
        setError({
          message:
            err.message || "Could not fetch places, please try again later.",
        });
      }
      setIsFetching(false);
    }
    fetchPlaces();
  }, []);
  if (error) {
    return <Error title="An error occurred" message={error.message} />;
  }
  return ( JSX );
}
```

## Get User Position (Transforming the Fetched Data)

Here, we cannot await the **getCurrentPosition** method. Instead, we need to pass the callback function which will be called once the position data arrives.

```
  useEffect(() => {
    async function fetchPlaces() {
      setIsFetching(true);
      try {
        const res = await fetch("http://localhost:3000/places");
        const resData = await res.json();
        if (!res.ok) {
          throw new Error("Failed to fetch places");
        }
        navigator.geolocation.getCurrentPosition((position) => {
          const sortedPlaces = sortPlacesByDistance(
            resData.places,
            position.coords.latitude,
            position.coords.longitude
          );
          setAvailablePlaces(sortedPlaces);
          setIsFetching(false);
        });
      } catch (err) {
        setError({
          message:
            err.message || "Could not fetch places, please try again later.",
        });
        setIsFetching(false);
      }
    }

    fetchPlaces();
  }, []);
```

## Extracting Code and Improving Code Structure

Create **http.js** file inside the _src_ folder. We just take this fetching the places logic into here.

```
export async function fetchAvailablePlaces() {
  const res = await fetch("http://localhost:3000/places");
  const resData = await res.json();
  if (!res.ok) {
    throw new Error("Failed to fetch places");
  }
  return resData.places;
}
```

## Sending Data

Inside the _http.js_ file =>

```
export async function updateUserPlaces(places) {
  const response = await fetch("http://localhost:3000/user-places", {
    method: "PUT",
    body: JSON.stringify({ places }),
    headers: {
      "Content-Type": "application/json",
    },
  });
  const resData = await response.json();
  if (!response.ok) {
    throw new Error("Failed to update user data.");
  }
  return resData.message;
}
```

Then we just use it to update the selected places.

```
  async function handleSelectPlace(selectedPlace) {
    setUserPlaces((prevPickedPlaces) => {
      if (!prevPickedPlaces) {
        prevPickedPlaces = [];
      }
      if (prevPickedPlaces.some((place) => place.id === selectedPlace.id)) {
        return prevPickedPlaces;
      }
      return [selectedPlace, ...prevPickedPlaces];
    });
    try {
      await updateUserPlaces([selectedPlace, ...userPlaces]);
    } catch (error) {
      // ...
    }
  }
```

## Optimistic Updating

Here, when we catch an error, we roll back to the previous state.

```
const [userPlaces, setUserPlaces] = useState([]);
const [errorUpdatingPlace, setErrorUpdatingPlaces] = useState();
…
  async function handleSelectPlace(selectedPlace) {
    setUserPlaces((prevPickedPlaces) => {
      if (!prevPickedPlaces) {
        prevPickedPlaces = [];
      }
      if (prevPickedPlaces.some((place) => place.id === selectedPlace.id)) {
        return prevPickedPlaces;
      }
      return [selectedPlace, ...prevPickedPlaces];
    });
    try {
      await updateUserPlaces([selectedPlace, ...userPlaces]);
    } catch (error) {
      setUserPlaces(userPlaces);
      setErrorUpdatingPlaces({
        message: error.message || "Failed to update places.",
      });
    }
  }
```

We could also do the fetching before updating the state, but in this case, we need to show a loading spinner or the user might think the app is stuck.

Then make sure to show the error to the user. Here, we use the modal.

```
      <Modal open={errorUpdatingPlace} onClose={handleError}>
        {errorUpdatingPlace && (
          <PageError
            title="An error occurred!"
            message={errorUpdatingPlace.message}
            onConfirm={handleError}
          />
        )}
      </Modal>
```

Then handleError is just a function to clear the error.

```
  function handleError() {
    setErrorUpdatingPlaces(null);
  }
```

## Deleting Data

Here, we just use the _updateUserPlaces_ function and send the filtered data. Then we also do optimistic updating when an error occurs.

```
  const handleRemovePlace = useCallback(
    async function handleRemovePlace() {
      setUserPlaces((prevPickedPlaces) =>
        prevPickedPlaces.filter(
          (place) => place.id !== selectedPlace.current.id
        )
      );
      try {
        await updateUserPlaces(
          userPlaces.filter((place) => place.id !== selectedPlace.current.id)
        );
      } catch (error) {
        setUserPlaces(userPlaces);
        setErrorUpdatingPlaces({
          message: error.message || "Failed to delete place.",
        });
      }
      setModalIsOpen(false);
    },
    [userPlaces]
  );
```
