Wynik (Boolean),Testowany ciąg (UUID),Opis przypadku

True,550e8400-e29b-41d4-a716-446655440000,Standardowy UUID v4 (małe litery)

True,6EC0BD7F-11C0-43DA-975E-2A8AD9EB0D83,Standardowy UUID v4 (wielkie litery)

True,00000000-0000-0000-0000-000000000000,"Nil UUID (pusty, ale poprawny technicznie)"

True,123e4567-e89b-12d3-a456-426614174000,Starszy UUID v1

False,550e8400-e29b-41d4-a716-44665544000,Błąd: Za krótki (brakuje jednego znaku na końcu)

False,z50e8400-e29b-41d4-a716-446655440000,Błąd: Niedozwolony znak 'z' (poza zakresem hex)

False,550e8400e29b41d4a716446655440000,Błąd: Brak myślników

False,550e8400-e29b-41d4-a716-446655440000-1,Błąd: Za długi (dodatkowy znak)

False,550e8400-e29b-41d4-a716-44665544000g,Błąd: Znak 'g' na końcu

False,550e8400--e29b-41d4-a716-446655440000,Błąd: Podwójny myślnik
