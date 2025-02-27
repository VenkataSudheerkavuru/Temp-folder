<div class="d-flex flex-column justify-content-start">
  <!-- Contact Details -->
  <div>
    <div class="d-flex justify-content-between">
      <div class="text-start">
        <h5 style="font-weight: 400; font-size: 25px;">{{ selectedContact.name }}</h5>
      </div>
      <div>
        <button class="btn btn-link" (click)="editContact(selectedContact)">Edit</button>
        <button class="btn btn-link" (click)="deleteContact(selectedContact)">Delete</button>
      </div>
    </div>
    <div class="text-start">
      <p>Email: {{ selectedContact.email }}</p>
      <p>Mobile: {{ selectedContact.mobile }}</p>
      <p>Landline: {{ selectedContact.landLine }}</p>
      <p>
        Website: 
        <a [href]="selectedContact.website" target="_blank">{{ selectedContact.website }}</a>
      </p>
      <p>Address: {{ selectedContact.address }}</p>
    </div>
  </div>
</div>
